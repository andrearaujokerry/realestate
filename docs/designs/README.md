# Job-Change Lead Engine — design options

Three ways to build the system described in the [requirements doc](<../../LinkedIn Job-Change Lead Engine — Requirements.pdf>) (Oct 3, 2026). All three meet the same requirements. They differ in where business logic lives, what runs the pipeline, and how many moving parts you have to operate.

> **Decision (Oct 3, 2026): Design B.** The build plan is in the [workplan](../workplan.md). It estimates the MVP bottom-up at ~76 engineer-days, with Gate 1 around week 10 for two engineers. That's longer than the top-down figures below, which were only meant for comparing the designs.

| | [A · Postgres-as-Backend](design-a-postgres-as-backend.md) | [B · Typed Monolith](design-b-typed-monolith.md) | [C · Two Planes](design-c-two-planes.md) |
|---|---|---|---|
| Thesis | The database is the backend | One TypeScript app with durable jobs (the spec's stack) | The pipeline is its own data product |
| Stack | Supabase (Postgres/PostGIS, Auth, Queues, Cron, Realtime, Vault), one TypeScript worker, React SPA | Next.js + tRPC + Drizzle on Vercel, Neon Postgres/PostGIS, Inngest | Python + Dagster data plane, B's Next.js app as the product plane, Postgres/PostGIS, object storage |
| Logic lives in | SQL (search, scoring, permissions, budgets) and the worker | TypeScript, with RLS underneath | Python for the pipeline, TypeScript for the product |
| You operate | One platform and one worker | Three or four managed services | Two codebases and five or six services |
| Phase 1 gate, est. | 3–5 weeks | 4–6 weeks (the spec's estimate) | 6–9 weeks |
| Infra per month, est. | $40–110 | $60–200 | $150–300 |
| Main risk | Logic in SQL; platform coupling | Vendor sprawl; tenant context relies on discipline | Two languages and two deploys; schema drift between them |
| Best for | A solo builder proving the thesis | A small TypeScript team heading toward multi-brokerage SaaS | Data-heavy scale: licensed bulk feeds, many metros |

The estimates assume one or two engineers. Use them to compare the designs, not as commitments. Infra excludes data licensing, which is the cost that decides the $500/month budget. It also excludes LLM spend, which is about $50/month in every design ([cost model](#llm-cost-model)). Vendor prices are rough, so check current pricing before deciding.

## Where they differ on the hard requirements

| Requirement | A · Postgres-as-Backend | B · Typed Monolith | C · Two Planes |
|---|---|---|---|
| Tenant isolation (SEC-3) | RLS is the only path to data | RLS under a per-request tenant transaction | As B; pipeline sets the tenant per partition |
| Admin MFA (SEC-1) | RLS requires the JWT's `aal2` claim | Middleware requires a TOTP-verified session | As B |
| Request budgets (ING-8) | SQL counter gates every request | Inngest throttle paces; SQL counter caps | SQL counter caps; per-source concurrency limit |
| Source kill switch (SRC-5) | Flag checked before every request | Flag check; `cancelOn` drops queued work | Flag check; run terminated |
| Dead letters (ING-14) | pgmq read count over 5 | Inngest `onFailure` handler | Failed partitions |
| Re-extraction (EXT-8) | Re-enqueue raw posts | Fan-out event | Bump `code_version`, backfill unsynced partitions |
| Rescoring (SCO-3, SCO-4) | One set-based `UPDATE` | Batched job | Rematerialize the `scores` asset |
| Scorer can't see protected fields | Composite input type; function owner can't read tables | Strict Zod input; no DB imports | Pydantic model with `extra="forbid"` |
| Person dedup (ING-13) | Profile URL, then trigram match | As A, in TypeScript | Splink probabilistic linkage |
| LLM calls | Synchronous | Synchronous | Message Batches with a synchronous fallback |
| Raw payloads (ING-11, DB-7) | Postgres, nightly purge | Postgres, nightly purge | Object storage, lifecycle expiry |
| New-leads pill (FEED-11) | Realtime push | 60-second poll | 60-second poll |

## What all three share

The spec already settles most of the domain, so these parts are the same in every design. The design docs don't repeat them.

### Sources

- Every design implements the spec's `SourceAdapter` contract (SRC-1 to SRC-6). Each launches with the spec's recommended sources: CSV import, a business-press adapter, and one licensed people-data provider. Choosing that provider is the Phase 0 gate, and the choice doesn't depend on architecture. No design scrapes LinkedIn directly.
- **Structured sources skip the LLM.** Licensed providers deliver records that are already structured: person, old and new employer, title, location, change date. Running those through classification and extraction costs money and loses precision. So `RawPost` gets an optional `structured` block, and the pipeline sends those items straight to enrichment with the confidence the source asserts. Adding a source still changes nothing downstream (SRC-4).

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
| `lead_sources` | Provenance that outlives the 90-day raw purge: source, URL, fetch time, run, terms version, extractor version (AUD-3) |
| `lead_markets` | Precomputed lead → market membership. "Agents see their market" (SEC-2) becomes an indexed equality check inside RLS, not a spatial test per row |
| `scoring_configs`, `score_audit` | Versioned weights; scoring inputs and outputs kept for the life of the lead (SCO-7, AUD-2) |
| `source_budgets`, `source_health` | Per-source request counters and circuit-breaker state (ING-8, ING-9) |
| `dead_letters` | Failed items with the error and attempt count, retryable from the admin UI (ING-14) |
| `crm_links` | Lead ↔ CRM record, last pushed hash, last synced status (INT-3, PRIV-3) |
| `outbox` | Side effects (CRM push, webhook, alert), written in the same transaction as the change that caused them |

Two schema consequences of the spec:

- PRIV-6 forbids cross-tenant pooling, so `raw_posts` belongs to a tenant. DB-5's unique key therefore becomes `(tenant_id, source, external_post_id)`.
- `raw_posts` stays unpartitioned so that key can be enforced. Postgres only enforces a unique constraint on a partitioned table if it includes the partition key. The 90-day purge (DB-7) is a nightly batched `DELETE`, which is trivial at a million rows a year. The append-only `audit_log` and `score_audit` tables are partitioned by month.

Tenant isolation goes in at Phase 1, even though the spec schedules roles (RBAC) for Phase 2. Adding `tenant_id` and RLS to every table later is the expensive part.

### Compliance machinery

All of this is MVP scope in every design:

- **Suppression.** The list holds keyed HMAC-SHA256 hashes of the normalized profile URL and of the normalized name plus metro. The spec says "salted hashes," but a keyed HMAC with its key in the secret store is the safer reading. Names and profile URLs are easy enough to guess that a hash stored next to its salt can be brute-forced. The list is checked at ingest (PRIV-5) and again before every export, CRM push, digest and webhook (INT-8).
- **Hard delete** (DB-8, PRIV-2) is one transactional procedure:
  - it removes the person's leads, raw posts, enrichments, notes, events, outreach log, score audit and CRM links
  - it writes the suppression hash
  - it flags every CRM the person was pushed to (PRIV-3)
  - it writes one audit row

  Audit rows hold ids, never personal data, so the trail survives the delete.
- **Phone numbers** come only from licensed sources. They're column-encrypted (SEC-5) and DNC-scrubbed before display (INT-9).

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

At the spec's volume of 50,000 raw posts a month:

| Stage | Calls/month | Tokens per call | Cost/call | Monthly |
|---|---|---|---|---|
| Regex gate keeps ≤ 20% | — | — | — | $0 |
| Haiku 4.5 classify ($1 / $5 per MTok) | 10,000 | ~1,500 in, ~50 out | ~$0.0018 | ~$18 |
| Sonnet extract ($2 / $10 per MTok), assuming 40% are real job changes | 4,000 | ~3,000 in, ~600 out | ~$0.012 | ~$48 |
| **Synchronous total** | | | | **~$65** |
| With prompt caching on the extraction prefix | | | | ~$48 |
| Through the Message Batches API (50% off, stacks with caching) | | | | ~$25 |

That's under the spec's own estimate of about $100. Either way, LLM spend is small next to data licensing: the Phase 0 vendor decides whether you stay under $500 a month, not the architecture. The classifier prompt is shorter than Haiku 4.5's 4,096-token caching minimum, so it doesn't get cached.

The spec names Claude Sonnet 5. Sonnet 5.5 (`claude-sonnet-5-5`) is now current at the same price. The model id is part of `extractor_version` and every change passes the golden-set gate, so you can start on 5.5, or stay on 5 and swap later at little cost.

## Tensions in the requirements to settle now

These came up while working through the designs. They apply to every option.

1. **The 90-day raw purge limits three other requirements.** Re-extraction (EXT-8) only reaches raw posts still inside the window. Provenance (AUD-3) has to be copied to `lead_sources` before the purge. And the card's two-line excerpt disappears unless it's stored on the lead. Decide whether a short excerpt may outlive the raw post.
2. **Seniority may invite an age-proxy argument.** Some state and local fair-housing laws protect more classes than the FHA's seven, and some of them include age. Seniority band (weight 15) is the scoring input most exposed to that argument. Ask counsel about it specifically.
3. **Score-audit volume.** Nightly recompute (SCO-4) combined with "every scoring run writes its inputs and output" (SCO-7) could mean 100,000 rows a night once the corpus is full. The designs log a row only when the inputs, config version or output change. The scorer is deterministic and versioned, so every ranking can still be reconstructed. Confirm that reading with counsel.
4. **Suppression key and scope.** Hashing the profile URL is precise, but press releases have no profile URL. Hashing name plus metro covers them, but can suppress someone else with the same name. Separately: should an opt-out received by one tenant apply to every tenant?
5. **Per-tenant copies multiply cost.** Under PRIV-6, ten tenants watching Austin each pay to classify the same press release. That's fine at MVP; at SaaS scale it becomes a pricing input.
6. **Map jitter vs. "never imply a home address."** MAP-5's jitter has to happen in screen space (see above). Put that in the spec so a later change doesn't bring back geographic offsets.

## Delivery

The spec's phases and gates hold for all three designs:

- **Phase 0:** the source decision plus legal review. In every design, the first code is the CSV adapter and the raw-post store, so the pipeline has data before a vendor is signed.
- **Phase 1:** one source runs through extraction to a feed sorted by recency, the map, and the review queue. No scoring yet.
- **Phase 2:** adds scoring, filters, saved views, the pipeline tab, the CRM connector, digests and roles.

The designs change how much effort each phase takes, not what's in it.
