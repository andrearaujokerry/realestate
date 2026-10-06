# Job-Change Lead Engine — design options

Three ways to build the system described in the [requirements doc](<../../LinkedIn Job-Change Lead Engine — Requirements.pdf>) (Oct 3, 2026). All three meet the same requirements. They differ in where business logic lives, what runs the pipeline, and how many moving parts you have to operate.

**Ingestion is manual (revised Oct 6, 2026).** You copy posts from LinkedIn and paste them into the engine. That replaces the automated sources (licensed data, press feeds) in all three designs. Everything downstream of the raw-post store is unchanged. See [Ingestion: manual paste](#ingestion-manual-paste).

> **Decision (Oct 3, 2026): Design B.** The build plan is in the [workplan](../workplan.md). It estimates the MVP bottom-up at ~71 engineer-days, with Gate 1 around week 9 for two engineers. That's longer than the top-down figures below, which were only meant for comparing the designs.

| | [A · Postgres-as-Backend](design-a-postgres-as-backend.md) | [B · Typed Monolith](design-b-typed-monolith.md) | [C · Two Planes](design-c-two-planes.md) |
|---|---|---|---|
| Thesis | The database is the backend | One TypeScript app with durable jobs (the spec's stack) | The pipeline is its own data product |
| Stack | Supabase (Postgres/PostGIS, Auth, Queues, Cron, Realtime, Vault), one TypeScript worker, React SPA | Next.js + tRPC + Drizzle on Vercel, Neon Postgres/PostGIS, Inngest | Python + Dagster data plane, B's Next.js app as the product plane, Postgres/PostGIS, object storage |
| Logic lives in | SQL (search, scoring, permissions) and the worker | TypeScript, with RLS underneath | Python for the pipeline, TypeScript for the product |
| You operate | One platform and one worker | Three or four managed services | Two codebases and five or six services |
| Phase 1 gate, est. | 2–4 weeks | 3–5 weeks | 5–8 weeks |
| Infra per month, est. | $40–110 | $60–200 | $150–300 |
| Main risk | Logic in SQL; platform coupling | Vendor sprawl; tenant context relies on discipline | Two languages and two deploys; schema drift between them |
| Best for | A solo builder proving the thesis | A small TypeScript team heading toward multi-brokerage SaaS | Only if you add bulk automated sources later; overbuilt for manual paste |

The estimates assume one or two engineers. Use them to compare the designs, not as commitments. With manual ingestion there's no data licensing, and LLM spend is about $15/month in every design ([cost model](#llm-cost-model)), so infra is now the main running cost. Vendor prices are rough, so check current pricing before deciding.

## Where they differ on the hard requirements

| Requirement | A · Postgres-as-Backend | B · Typed Monolith | C · Two Planes |
|---|---|---|---|
| Paste intake | Browser segments; `submit_paste()` RPC stores the run | Browser segments; `ingest.submit` mutation stores the run | As B; a sensor turns each paste into a partition |
| Segmenter's model fallback | Edge Function calls Haiku | tRPC procedure calls Haiku | As B |
| Paste progress | Realtime push | Polled every few seconds | Polled, after up to 30 s of sensor delay |
| Tenant isolation (SEC-3) | RLS is the only path to data | RLS under a per-request tenant transaction | As B; pipeline sets the tenant per partition |
| Admin MFA (SEC-1) | RLS requires the JWT's `aal2` claim | Middleware requires a TOTP-verified session | As B |
| Dead letters (ING-14) | pgmq read count over 5 | Inngest `onFailure` handler | Failed partitions |
| Re-extraction (EXT-8) | Re-enqueue raw posts | Fan-out event | Bump `code_version`, backfill unsynced partitions |
| Rescoring (SCO-3, SCO-4) | One set-based `UPDATE` | Batched job | Rematerialize the `scores` asset |
| Scorer can't see protected fields | Composite input type; function owner can't read tables | Strict Zod input; no DB imports | Pydantic model with `extra="forbid"` |
| Person dedup (ING-13) | Profile URL, then trigram match | As A, in TypeScript | Splink probabilistic linkage |
| LLM calls | Synchronous | Synchronous | Synchronous for pastes; Batches for backfills |
| Raw payloads (ING-11, DB-7) | Postgres, nightly purge | Postgres, nightly purge | Object storage, lifecycle expiry |
| New-leads pill (FEED-11) | Realtime push | 60-second poll | 60-second poll |

## What all three share

The spec already settles most of the domain, so these parts are the same in every design. The design docs don't repeat them.

### Ingestion: manual paste

You add posts by hand. The engine's job is to make that fast, and to turn whatever you paste into clean, deduplicated raw posts. From the raw-post store onward, the pipeline is unchanged.

**What to paste.** Copy straight from LinkedIn and paste into the engine's paste box. That can be one post, or a whole page of search results or feed: select, copy, paste. A paste can hold up to 500 posts. Other formats work too:

| Format | When to use it |
|---|---|
| Copy from LinkedIn, paste into the box (recommended) | Everyday use. The browser puts both text and HTML on the clipboard, and the HTML carries the links behind names, so author profile URLs come along for free |
| CSV upload: `post_url, author_name, author_profile_url, posted, text, location_hint` | Posts you've collected in a spreadsheet |
| Plain text, posts separated by a line containing only `---`, with the post link on the first line if you have it | Posts collected in notes |

Don't type the extracted fields (title, company, start date) yourself. That's the extractor's job; typing them is slower and adds errors, and the review queue is where corrections belong. Two pieces of metadata *are* worth adding:

- **The post link** ("Copy link to post"), when you paste single posts. It gives a stable id for deduplication, the provenance URL, and the card's link to the original.
- **A location hint for the batch**, such as "Austin" when everything in the paste came from an Austin-focused search. It fills SRC-2's "observed location hint" on `RawPost`. The extractor uses it only when the post doesn't name a city, and marks those locations as lower confidence.

**The segmenter** is one TypeScript module. It runs in the browser for an instant preview and again on the server at submit. It:

1. Drops LinkedIn's interface text: Like, Comment, Repost, Send, Follow, reaction and comment counts, "…more", Promoted. Dropping comment lines also keeps commenters' names out of storage.
2. Splits on post headers: name, connection degree, headline, relative time ("2d •").
3. Pulls profile links (`/in/…`) and post links (activity ids) from the clipboard HTML.
4. Resolves relative times ("2d", "3w", "1mo") against the paste time, and records their precision.
5. Falls back to Haiku 4.5 when its rules can't find clean boundaries. Haiku gets the numbered lines and returns each post's line range; code then slices the original text, so a model never retypes a post.

**Preview, then submit.** The parsed posts appear in a table showing author, time, first line, captured links, and whether the regex gate thinks it's a job change. Uncheck junk, merge or split mis-cut posts, then submit.

**Store.** Each submit is one row in the spec's `source_runs` table: who pasted, when, the batch hints, and counts that update as posts move through the pipeline (ING-15). Each post becomes a `RawPost` with source `manual_paste`, stored verbatim with provenance (ING-11).

- The dedup key is the LinkedIn activity id when the paste carried the post link. Otherwise it's a hash of the normalized author and text (ING-12). Re-pasting overlapping pages is harmless (ING-7).
- Suppressed people are dropped at submit (PRIV-5).
- You can undo a paste. Deleting a run removes its raw posts, plus anything derived from them that nobody has worked yet.

**Who can paste.** Admins, by default. A tenant setting lets agents paste too.

**What's gone.** Everything that existed to fetch data automatically:

- scheduled runs and cursors (ING-6)
- request budgets (ING-8)
- backoff and the circuit breaker (ING-9)
- robots.txt handling (ING-10)
- source credentials (SRC-6, SEC-7)
- the zero-results alert (ING-16)
- the licensed and press adapters

The `SourceAdapter` and `RawPost` contract stays as the seam (SRC-1, SRC-2, SRC-4), with manual paste as its only adapter. An automated source can still be added later without touching anything downstream, and the machinery above comes back with it.

**Knock-on effects.** Nothing downstream changes, but three things behave differently:

- **Locations are often missing.** Posts rarely name a city, and a copied post header shows no location. Expect more records without `new_location`, which EXT-7 already routes to review. The batch hint helps, and reviewers can pick the city by hand.
- **Post dates are approximate.** "2w" means 14 days, give or take 3. Recency scoring decays over 60 days, so that's tolerable.
- **Volume and freshness follow you.** Posts are processed within minutes of a paste. The spec's 24-hour freshness target now depends on how often you paste.

**A legal note.** The spec rates manual import as no legal risk, and pasting into your own tool is far from the automated scraping LinkedIn has sued over. But LinkedIn's User Agreement restricts copying members' content in general, not only with bots.

- It's a contract question, and the live risk is to your LinkedIn account.
- Keep the copying genuinely manual: no extensions, macros or auto-scrollers.
- Keep this first on counsel's list.
- Your privacy duties don't change. Once you store the data, you're its controller.

### Pipeline

The stages are the spec's: regex gate → Haiku 4.5 classifier → Sonnet structured extraction → enrichment → scoring. Each design adds three things:

- **Structured outputs, not forced tool calls.** Extraction passes the JSON schema through `output_config.format`. Sonnet 5.5 rejects a forced `tool_choice`.
- **Evidence spans.** Every extracted field comes back with the words in the post that support it. Code checks that the span appears verbatim in the post and nulls the field if it doesn't. That enforces EXT-5's "null rather than guess" instead of trusting the model to follow it.
- **Calibrated confidence.** Self-reported LLM confidence tends to cluster high, so an uncalibrated 0.75 threshold (EXT-7) might route almost nothing to review. Raw confidences are mapped to observed accuracy on the golden set first. The calibrated values drive review routing and the unverified cap (SCO-6).

Any change to a prompt, schema or model runs the golden-set eval in CI before it can ship (EXT-9). The model id is part of `extractor_version`.

### Scoring

`score(input, weights, now) → { score, factors }` is a pure, deterministic function. Its input type contains only the six signal inputs plus extraction confidence. There is no field for name, photo, school or demographics, which is what the spec means by "enforced at the schema level." Weights are a versioned `scoring_configs` row per tenant. Each lead stores the `score_version` that ranked it.

### Data model

The spec's ten core entities, plus:

| Table | Purpose |
|---|---|
| `lead_sources` | Provenance that outlives the 90-day raw purge: source, post URL when captured, paste time, run, who pasted, extractor version (AUD-3) |
| `lead_markets` | Precomputed lead → market membership. "Agents see their market" (SEC-2) becomes an indexed equality check inside RLS, not a spatial test per row |
| `scoring_configs`, `score_audit` | Versioned weights; scoring inputs and outputs kept for the life of the lead (SCO-7, AUD-2) |
| `dead_letters` | Items that failed segmentation or extraction, with the error and attempt count, retryable from the admin UI (ING-14) |
| `crm_links` | Lead ↔ CRM record, last pushed hash, last synced status (INT-3, PRIV-3) |
| `outbox` | Side effects (CRM push, webhook, alert), written in the same transaction as the change that caused them |

Each paste is a row in the spec's own `source_runs` table, with who pasted, the batch hints, and per-stage counts.

Two schema consequences of the spec:

- PRIV-6 forbids cross-tenant pooling, so `raw_posts` belongs to a tenant. DB-5's unique key therefore becomes `(tenant_id, source, external_post_id)`. For pasted posts, `external_post_id` is the LinkedIn activity id or the content hash.
- `raw_posts` stays unpartitioned so that key can be enforced. Postgres only enforces a unique constraint on a partitioned table if it includes the partition key. The 90-day purge (DB-7) is a nightly batched `DELETE`, which is trivial at a million rows a year. The append-only `audit_log` and `score_audit` tables are partitioned by month.

Tenant isolation goes in at Phase 1, even though the spec schedules roles (RBAC) for Phase 2. Adding `tenant_id` and RLS to every table later is the expensive part.

### Compliance machinery

All of this is MVP scope in every design:

- **Suppression.** The list holds keyed HMAC-SHA256 hashes of the normalized profile URL (captured from the paste's HTML when present) and of the normalized name plus metro. The spec says "salted hashes," but a keyed HMAC with its key in the secret store is the safer reading. Names and profile URLs are easy enough to guess that a hash stored next to its salt can be brute-forced. The list is checked when you submit a paste (PRIV-5), and again before every export, CRM push, digest and webhook (INT-8).
- **Hard delete** (DB-8, PRIV-2) is one transactional procedure:
  - it removes the person's leads, raw posts, enrichments, notes, events, outreach log, score audit and CRM links
  - it writes the suppression hash
  - it flags every CRM the person was pushed to (PRIV-3)
  - it writes one audit row

  Audit rows hold ids, never personal data, so the trail survives the delete.
- **Phone numbers.** Pasted posts don't carry any. If a later source supplies them, they're column-encrypted (SEC-5) and DNC-scrubbed before display (INT-9).

### Geography and the map

- **Gazetteer first.** Resolution is capped at a city or metro centroid (ENR-2), so geocoding is mostly a lookup in an open gazetteer: US Census place centroids plus Census metro-area (CBSA) definitions, or GeoNames. A commercial geocoder sits behind the provider interface (ENR-1) as a fallback for ambiguous strings. This has three benefits:
  - Lookups cost nothing.
  - It avoids the limits some commercial geocoders put on storing results.
  - It hands relocation detection (ENR-5) the metro codes it compares.

  And the system holds no street-level data that could leak.
- **Pins never look like an address.** Leads that share a centroid fan out in screen space around it, and the maximum zoom stops at city level. Offsetting pins by meters on the ground would put each one on a specific block at street zoom, which implies a home address (MAP-1).
- **One query, two projections** (MAP-6). One filter compiler produces one predicate.
  - The feed projects cards, using keyset pagination on `(score DESC, detected_at DESC, id)`, which is the DB-2 index.
  - The map projects compact `(id, location_id, band, status)` tuples inside the viewport, plus a small location dictionary. 5,000 pins come to about 50 KB gzipped.

  Filter state lives in the URL (MAP-9).

### LLM cost model

At about 3,000 pasted posts a month (100 a day):

| Stage | Calls/month | Tokens per call | Cost/call | Monthly |
|---|---|---|---|---|
| Segmenter fallback, Haiku 4.5 (about one call per large paste) | ~60 | ~15,000 in, ~500 out | ~$0.018 | ~$1 |
| Regex gate keeps ~50% of a hand-picked paste | — | — | — | $0 |
| Haiku 4.5 classify ($1 / $5 per MTok) | 1,500 | ~1,500 in, ~50 out | ~$0.0018 | ~$3 |
| Sonnet extract ($2 / $10 per MTok), assuming 70% are real job changes | ~1,000 | ~3,000 in, ~600 out | ~$0.012 | ~$12 |
| **Total** | | | | **~$16** |

Prompt caching on the extraction prefix brings that to about $12. There's nothing to batch, because someone is waiting on each paste. Data licensing drops to zero, so the $500/month budget now covers only infrastructure. The classifier prompt is shorter than Haiku 4.5's 4,096-token caching minimum, so it doesn't get cached.

The spec names Claude Sonnet 5. Sonnet 5.5 (`claude-sonnet-5-5`) is now current at the same price. The model id is part of `extractor_version` and every change passes the golden-set gate, so you can start on 5.5, or stay on 5 and swap later at little cost.

## Tensions in the requirements to settle now

These came up while working through the designs. They apply to every option.

1. **LinkedIn's User Agreement vs. copying by hand.** See the [legal note](#ingestion-manual-paste) above. Get counsel's view before you paste at volume.
2. **The 90-day raw purge limits three other requirements.** Re-extraction (EXT-8) only reaches raw posts still inside the window. Provenance (AUD-3) has to be copied to `lead_sources` before the purge. And the card's two-line excerpt disappears unless it's stored on the lead. Decide whether a short excerpt may outlive the raw post.
3. **Seniority may invite an age-proxy argument.** Some state and local fair-housing laws protect more classes than the FHA's seven, and some of them include age. Seniority band (weight 15) is the scoring input most exposed to that argument. Ask counsel about it specifically.
4. **Score-audit volume.** Nightly recompute (SCO-4) combined with "every scoring run writes its inputs and output" (SCO-7) could mean 100,000 rows a night once the corpus is full. The designs log a row only when the inputs, config version or output change. The scorer is deterministic and versioned, so every ranking can still be reconstructed. Confirm that reading with counsel.
5. **Suppression key and scope.** Profile URLs come from the clipboard HTML, but a plain-text or CSV paste may lack them. Hashing name plus metro covers those, but can suppress someone else with the same name. Separately: should an opt-out received by one tenant apply to every tenant?
6. **Map jitter vs. "never imply a home address."** MAP-5's jitter has to happen in screen space (see above). Put that in the spec so a later change doesn't bring back geographic offsets.

## Delivery

The spec's phases and gates hold for all three designs, adjusted for manual ingestion:

- **Phase 0:** counsel review of manual collection and the privacy notice. There's no data vendor to choose. In every design, the first code is the paste box and the raw-post store, so you can start pasting real posts in week 3 or 4.
- **Phase 1:** your pastes run through extraction to a feed sorted by recency, the map, and the review queue. No scoring yet.
- **Phase 2:** adds scoring, filters, saved views, the pipeline tab, the CRM connector, digests and roles.

The designs change how much effort each phase takes, not what's in it.
