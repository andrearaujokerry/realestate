# Workplan — Design B (Typed Monolith)

This plan covers building the MVP of the [Job-Change Lead Engine](<../LinkedIn Job-Change Lead Engine — Requirements.pdf>) on [Design B](designs/design-b-typed-monolith.md), and what comes after it. "MVP" here means the spec's Phase 0 and Phase 1. It ends at the spec's Phase 1 gate: **an agent would call 10 of the first 50 leads.**

## At a glance

| | |
|---|---|
| Engineering effort | ~76 engineer-days (realistic range 60–90) |
| Your effort | ~11 days: choosing the market, vendors, counsel, golden-set labeling, the pilot. Counsel's own time is extra |
| Two engineers | ~8 weeks to build, then a 2-week pilot. Gate 1 around week 10: mid-December 2026 if you start Oct 5 |
| One engineer | ~14 weeks to build, then the pilot. Gate 1 around week 16. With holidays that's early February 2027, or mid-to-late February if the same person also does Step 0 |
| Running cost during MVP | ~$60–200/month infra, $10–65/month LLM, plus data licensing ([cost model](designs/README.md#llm-cost-model)) |

This is longer than the spec's 5–8 weeks for Phases 0 and 1. It's also longer than the 4–6 weeks in the design doc, which reused the spec's top-down figure. Counted bottom-up, the work the spec itself makes MVP scope adds about 25 days that a 4–6 week Phase 1 doesn't have room for:

- compliance machinery
- tenant isolation
- the evaluation harness
- source rate limits and backoff
- launch hardening

A [fast track](#fast-track) cuts about 9 days.

**Assumptions behind the estimates:**

- One mid-to-senior TypeScript engineer per track, working five focused days a week.
- Tests and code review are included in each task.
- There's no discount for AI-assisted coding. Building with Claude Code tends to speed up implementation-heavy work: scaffolding, CRUD screens, tests. It doesn't speed up the calendar-bound work: counsel, vendor contracts, labeling, the pilot.

## MVP scope

**In:**

- One market and one live source, plus CSV import.
- The full interpretation pipeline: regex gate, Haiku classifier, Sonnet extraction, enrichment.
- A review queue for low-confidence extractions.
- A feed sorted by recency, with:
  - a detail drawer
  - statuses and notes
  - a few basic filters
  - CSV export
- A map with clustering, pin previews and the market overlay.
- Everything the spec puts in MVP for compliance:
  - suppression list
  - hard delete
  - subject-access lookup
  - audit log
  - provenance
  - privacy notice
  - 90-day purge of raw posts
- Tenant isolation with RLS, two roles (admin and agent), and MFA for admins.

**Out** (see the [backlog](#post-mvp-backlog)):

- scoring (the spec holds it back until you've seen real data)
- the filter rail and saved views
- the pipeline tab
- the CRM connector and digests
- the manager role and lead assignment
- bulk actions
- Slack and webhooks

## Decisions needed

| # | Decision | Owner | Needed by (two engineers / solo) | Recommendation |
|---|---|---|---|---|
| D1 | Launch metro and radius | You | Week 1 | A metro where one or two agents can run the pilot |
| D2 | Who the MVP is for | You | Week 1 | One team, set up as one tenant. The schema stays multi-tenant |
| D3 | Launch source (Gate 0) | You + counsel | Week 6 / week 12 | The licensed provider if its contract and counsel review are done in time. Otherwise business press, which needs no contract |
| D4 | Sonnet 5 or 5.5 for extraction | You | Week 3 / week 4 | 5.5. It's the same price, and the golden-set eval will confirm or reject it |
| D5 | Can a short post excerpt outlive the 90-day raw purge? | You + counsel | Week 7 / week 12 | Keep up to 200 characters on the lead if counsel agrees. Otherwise show only the link |
| D6 | Counsel sign-off on source terms, privacy notice, suppression scope and the CRM handoff | Counsel | Before the pilot | — |
| D7 | Pilot agents | You | Week 7 / week 13 | One or two agents who'll say honestly whether they'd call each lead |
| D8 | Which CRM you use | You | Before V1 | Decides the first connector (V1-7) |
| D9 | Seniority as a scoring input | Counsel | Before V1 scoring | See the age-proxy question in the [design notes](designs/README.md#tensions-in-the-requirements-to-settle-now) |

## Steps

Each step lists its tasks in build order. "Days" means engineer-days, and the "After" column lists the tasks that must finish first. Requirement IDs refer to the spec.

### Step 0 — Decide (Phase 0, your track)

This runs in weeks 1–2, alongside Step 1. Engineering doesn't need its result until Step 9.

| ID | Task | Your days |
|---|---|---|
| 0.1 | Pick the launch metro and radius. Line up pilot agents | 0.5 |
| 0.2 | Evaluate licensed providers. Shortlist three and get sample data for the metro. Compare coverage, freshness, fields, delivery format, price and terms (permitted use, indemnity, how they handle deletion requests) | 3 |
| 0.3 | List the press sources for the metro: business-journal sections, press-wire feeds, newsrooms of the largest employers. Check their terms and robots.txt, and estimate monthly volume | 1 |
| 0.4 | Counsel review of source terms, privacy notice, suppression scope and the CRM handoff | 1 + counsel time |
| 0.5 | Choose the launch source → **Gate 0** | 0.5 |
| 0.6 | Collect and label the golden set: 200 items, including hard negatives such as promotions, work anniversaries and hiring posts. Label the first 100 by week 3; the review queue adds the rest | 3 |
| 0.7 | Create accounts: GitHub, Vercel, Neon, Inngest, Anthropic (with a spend limit), AWS KMS, Resend, MapTiler, Sentry, plus a domain | 0.5 |
| 0.8 | Coordinate the pilot (Step 10) | 1.5 |

**Gate 0:** a data source is chosen and counsel has signed off on it.

### Step 1 — Foundation

| ID | Task | Days | After |
|---|---|---|---|
| 1.1 | Scaffold: Next.js App Router, strict TypeScript, tRPC, Drizzle, Tailwind with shadcn/ui, ESLint, Vitest, Playwright | 1 | — |
| 1.2 | CI on GitHub Actions. Each pull request gets a Vercel preview deploy and its own Neon database branch with migrations applied | 1 | 1.1 |
| 1.3 | Schema v1: the spec's core tables and enums, plus the shared additions from the design notes (`lead_sources`, `lead_markets`, `source_budgets`, `source_health`, `dead_letters`). Enable PostGIS and pg_trgm. Indexes DB-1 to DB-6 | 2.5 | 1.1 |
| 1.4 | Tenancy (SEC-3): an `app_user` database role that doesn't own the tables and can't bypass RLS; FORCE RLS; the policies; the tenant-scoped tRPC middleware on Neon's pooled driver; a lint ban on importing the root client; a cross-tenant test harness | 2 | 1.3 |
| 1.5 | Auth (SEC-1): Better Auth with magic links sent through Resend, organizations as tenants, admin and agent roles, TOTP required for admins | 2 | 1.4 |
| 1.6 | Credential vault: source credentials encrypted with a KMS key, with rotation (SEC-6, SEC-7) | 1 | 1.3 |
| 1.7 | Observability: Sentry on client, server and Inngest functions; structured logs that carry request and run ids | 0.5 | 1.1 |

**Done when** all three hold:

- A pull request deploys a preview with its own database branch.
- Magic-link login works.
- The cross-tenant test proves tenant A can't read tenant B's data through any procedure.

### Step 2 — Ingestion framework and CSV import

The spec asks for the CSV path in Phase 0, so the rest of the pipeline has data to run on before a live source exists.

| ID | Task | Days | After |
|---|---|---|---|
| 2.1 | The `SourceAdapter` contract; the `RawPost` Zod schema with its optional `structured` block; an adapter registry; terms and budget metadata per source (SRC-1 to SRC-4) | 1 | 1.3 |
| 2.2 | Inngest: typed events, the `/api/inngest` route, a 15-minute cron that starts runs when they're due, and `source_runs` status and counters (ING-6, ING-15) | 1.5 | 2.1 |
| 2.3 | Rate budgets and kill switch (ING-8, SRC-5): per-minute and per-day counters in Postgres, the Inngest throttle, a check of the enabled flag, and `cancelOn` to drop queued work when a source is disabled | 1 | 2.2 |
| 2.4 | HTTP wrapper (ING-9, ING-10): exponential backoff on 429 and 403 responses. After three failures in a row, a circuit breaker halts the run, disables the source and alerts an admin. Honors robots.txt for web sources | 1.5 | 2.2 |
| 2.5 | Raw store (ING-7, ING-11, ING-12, PRIV-5): insert with ON CONFLICT on `(tenant_id, source, external_post_id)`; a near-duplicate hash; a suppression-list check via keyed HMAC; the verbatim payload stored with provenance | 1 | 2.1 |
| 2.6 | Failure handling (ING-14, ING-16): failed jobs land in `dead_letters` via Inngest's `onFailure`. Email alerts go out when the breaker trips or a run returns zero new items | 1 | 2.2 |
| 2.7 | CSV adapter and an admin upload page: template, column mapping, validation, preview, and import as a run | 1.5 | 2.1, 1.5 |

**Done when:**

- Uploading the same CSV twice produces one set of raw posts, with provenance and a run record.
- A suppressed person in the file is dropped.

### Step 3 — Extraction

| ID | Task | Days | After |
|---|---|---|---|
| 3.1 | Regex gate (EXT-1): positive and negative patterns in a `patterns` table, seeded from ING-1 and ING-3. A separate matcher handles structured "started a new position" updates (ING-2). Logs how much it discards | 1 | 2.5 |
| 3.2 | Classifier on Haiku 4.5 (EXT-2 to EXT-4): structured output with `is_job_change`, a confidence score and the look-alike type. Posts below the confidence floor are dropped and logged | 1.5 | 3.1 |
| 3.3 | Extractor on Sonnet (EXT-5, EXT-6), passing the JSON schema through `output_config.format`. It returns a **list** of people per item, because press roundups name several. Each field gets a value, a confidence score and an evidence span (the supporting text). Unstated fields come back null, not guessed | 2 | 3.2 |
| 3.4 | Post-processing (EXT-7): check each evidence span against the text; resolve relative start dates ("starting in March") against the post date; normalize locations; compute overall confidence; route low-confidence records to review | 1.5 | 3.3 |
| 3.5 | Versioning (EXT-8): `extractor_version` is a hash of prompt, schema and model, stored on each lead. A script re-runs extraction over raw posts still inside the retention window | 1 | 3.3 |
| 3.6 | Evaluation harness (EXT-9, EXT-10): a golden-set format; a runner that reports precision, recall and per-field accuracy; a CI job that blocks any prompt, schema or model change that falls below target | 2 | 3.3, 0.6 |
| 3.7 | Calibrate each field's confidence scores against the golden set, stored as versioned config | 1 | 3.6 |
| 3.8 | Iterate on prompts until the targets are met: precision ≥ 0.95, recall ≥ 0.80, field accuracy ≥ 0.90 for title and company and ≥ 0.85 for location and start date | 3 | 3.6 |

**Done when:** the CI eval job meets every EXT-10 target on at least 200 labeled items.

### Step 4 — Enrichment and lead assembly

| ID | Task | Days | After |
|---|---|---|---|
| 4.1 | Gazetteer (ENR-1, ENR-2): load Census place centroids and metro-area (CBSA) definitions into `locations`. Add a normalizer for location strings such as "Greater Austin Area" and "Austin, Texas, United States", a cache, and a geocoder interface with a slot for a fallback provider | 2 | 1.3 |
| 4.2 | Relocation (ENR-5): compare the new and prior metros, fall back to the profile location, and mark it `unknown` otherwise | 0.5 | 4.1, 3.4 |
| 4.3 | Company resolution v1 (ENR-3): normalize the name and fuzzy-match it with trigrams. Take the domain and industry from the source when it supplies them | 0.5 | 1.3 |
| 4.4 | Person and lead dedup (ING-13): match on profile URL first, then on name, company and metro. Keep one lead per person per job change, linked to all its `lead_sources` | 1.5 | 4.3 |
| 4.5 | Market config (metro plus radius, ING-5) and precomputed `lead_markets` rows | 0.5 | 4.1 |
| 4.6 | Lead assembly: create or update the lead with its centroid, precision and provenance, and write a `lead_event` | 1 | 4.2, 4.4, 4.5 |

**Done when:** a CSV row about someone moving from Chicago to Austin becomes one lead. It sits at the Austin centroid, is marked as relocating from Chicago, and carries its provenance chain.

### Step 5 — Review queue

| ID | Task | Days | After |
|---|---|---|---|
| 5.1 | Admin review page (REV-1, REV-2, UI-8): a queue with a backlog badge, the post text beside the extracted fields, and inline edits. Reviewers can approve, correct and approve, or reject with a reason | 2 | 3.4, 1.5 |
| 5.2 | Corrections are saved to `extraction_labels`, which grows the golden set. Approved records become leads (REV-3) | 0.5 | 5.1, 4.6 |

**Done when:**

- An admin can clear the queue.
- Every correction shows up as a labeled example in the evaluation harness.

### Step 6 — Feed and detail drawer

| ID | Task | Days | After |
|---|---|---|---|
| 6.1 | App shell: layout, tabs, auth screens, responsive down to tablet width, theme. Quick wireframes for the card and the drawer | 1.5 | 1.5 |
| 6.2 | Lead card with:<br>• name and headline<br>• role and company<br>• location with the Relocating badge<br>• start date, absolute and relative<br>• inline status<br>• post excerpt and link<br>• actions: mark contacted, add note, dismiss<br>• Unverified marker<br>Company logos come later (V1-19) | 2 | 6.1 |
| 6.3 | `leads.page` on the shared filter compiler (FEED-2, FEED-3, FEED-6). Filters: relocations only, unverified only, date range, show dismissed. Keyset pagination by `detected_at`, infinite scroll 25 at a time, scroll position restored | 2 | 1.4, 6.2 |
| 6.4 | Detail drawer (UI-5, FEED-4, AUD-3): full post text, fields with their confidence, the provenance chain, a map inset, an activity timeline and notes | 2 | 6.3 |
| 6.5 | Card actions write `lead_events` and audit rows. "Contacted" records a channel in `outreach_log` (AUD-1, INT-7) | 1 | 6.3 |
| 6.6 | Skeleton cards while loading, the three empty states, and error states (FEED-8, FEED-9) | 0.5 | 6.3 |
| 6.7 | CSV export of the current view (INT-1, INT-8, SEC-4). Admin only. Suppression is re-checked, and each export is logged with its row count and filter | 1 | 6.3 |

**Done when:**

- An agent can work the feed top to bottom on a laptop or tablet without opening anything else.
- An admin can export what they see.

### Step 7 — Map

| ID | Task | Days | After |
|---|---|---|---|
| 7.1 | MapLibre with MapTiler or Protomaps tiles, loaded only when the Map tab opens | 1 | 6.1 |
| 7.2 | `leads.points` on the same filter compiler: limited to the visible map area, returning compact tuples plus a location dictionary (MAP-6, MAP-10) | 1 | 6.3 |
| 7.3 | Pins (MAP-1, MAP-2, MAP-3, MAP-5):<br>• clustered with Supercluster<br>• colored by relocation or status, with a legend<br>• pins that share a centroid fan out on screen with a count badge<br>• zoom capped at city level | 2 | 7.2 |
| 7.4 | Map extras (MAP-4, MAP-8, MAP-9, MAP-11):<br>• clicking a pin shows a preview that opens the drawer<br>• market overlay<br>• a reconciliation count ("412 leads, 389 mapped") linking to the unmapped ones<br>• filter state in the URL, shared with the feed | 1.5 | 7.3, 6.4 |

**Done when:**

- Switching between feed and map keeps the same result set.
- No zoom level shows a pin more precisely than a city.

### Step 8 — Compliance and admin

| ID | Task | Days | After |
|---|---|---|---|
| 8.1 | Person lookup by name or profile URL, showing everything held about them, with JSON export (PRIV-1) | 1 | 4.6 |
| 8.2 | Hard delete (DB-8, DB-9, PRIV-2): one transactional procedure that removes leads, raw posts, enrichments, notes, events and labels, writes the suppression hash, and leaves an audit row with no personal data. Admin screen with a confirmation step | 1.5 | 8.1 |
| 8.3 | Opt-out without deletion. A privacy notice page with an address for requests (PRIV-4, PRIV-5) | 0.5 | 2.5 |
| 8.4 | Audit middleware recording views, exports, edits and deletes, plus an admin audit viewer (AUD-1) | 1 | 1.4 |
| 8.5 | Nightly purge of raw posts older than 90 days. It runs after provenance has been copied to the lead, along with the excerpt if D5 allows it (DB-7) | 0.5 | 4.6 |
| 8.6 | Source health page (UI-9, ING-14, ING-15, ING-16): the last 50 runs per source with counts and status, a breaker reset, and failed items with retry | 1 | 2.6 |
| 8.7 | Source settings: enable or disable a source per tenant, enter its credentials (stored encrypted), set cadence and budget | 0.5 | 1.6, 2.3 |

**Done when:**

- Deleting a test person removes every trace of them.
- Re-importing that person is blocked by the suppression list.

### Step 9 — Live source

This starts as soon as Gate 0 closes. Move it earlier if you can, because real volume makes prompt tuning (3.8) easier to judge.

| ID | Task | Days | After |
|---|---|---|---|
| 9.1 | Adapter for the chosen source.<br>• Licensed: an API or bulk-delta client that maps records into `structured` RawPosts.<br>• Press: feeds and newsroom pages fetched with robots.txt respected, converted from HTML to text, with several people per item.<br>Either way it declares its terms and tracks an incremental cursor | 3.5 | Gate 0, 2.4 |
| 9.2 | Backfill 30–60 days for the launch metro. Tune patterns and budgets on real volume | 0.5 | 9.1 |

**Done when:** scheduled runs fetch, deduplicate and extract for three days with no manual help.

### Step 10 — Launch and pilot

| ID | Task | Days | After |
|---|---|---|---|
| 10.1 | Production setup: environment, domain, Resend SPF and DKIM, Neon point-in-time restore, production keys for Inngest and KMS, Sentry alerts | 1 | Steps 1–9 |
| 10.2 | Playwright smoke tests: login, feed, drawer, status change, export, review approval, deleting a person | 1.5 | Steps 5, 6, 8 |
| 10.3 | Performance test on 100,000 synthetic leads: feed p95 under 500 ms, map p95 under 800 ms, 5,000 pins drawn in under 1 second | 1 | Step 7 |
| 10.4 | Security and accessibility pass: cross-tenant tests, dependency audit, secret scan, security headers. Keyboard focus, contrast and labels on the feed | 1.5 | Steps 1–9 |
| 10.5 | Runbooks for handling a deletion request, rotating a credential, resetting the breaker, and re-running extraction | 0.5 | Step 8 |
| 10.6 | Pilot support: two weeks with one or two agents. Review the first 50 leads with them, and sort what they miss into fixes or the backlog | 2 | 10.1 |
| 10.7 | Buffer for bug fixes | 3 | — |

**Gate 1 (MVP done)** when all of these hold:

- The pilot agent would call at least 10 of the first 50 leads (the spec's gate).
- Golden-set metrics meet the EXT-10 targets on at least 200 items.
- One live source plus CSV ran on schedule for two weeks, with alerts working.
- Suppression, hard delete and export logging pass their tests.
- Feed and map meet their p95 targets at 100,000 leads.
- Counsel has signed off on the source and the privacy notice.

## Effort and schedule

| Step | Days | Two engineers | Solo |
|---|---|---|---|
| 0 Decide (your track) | 11 (yours) | Weeks 1–2; Gate 0 by week 6 at the latest | Weeks 1–2; Gate 0 by week 12 at the latest |
| 1 Foundation | 10 | Weeks 1–2 | Weeks 1–2 |
| 2 Ingestion + CSV | 8.5 | Weeks 2–3 | Weeks 3–4 |
| 3 Extraction | 13 | Weeks 3–5, tuning in week 7 | Weeks 4–7 |
| 4 Enrichment | 6 | Week 3 (gazetteer), weeks 5–6 | Weeks 7–8 |
| 5 Review queue | 2.5 | Weeks 5–6 | Week 8 |
| 6 Feed + drawer | 10 | Weeks 3–5 | Weeks 9–10 |
| 7 Map | 5.5 | Weeks 6–7 | Week 11 |
| 8 Compliance + admin | 6 | Weeks 7–8 | Week 12 |
| 9 Live source | 4 | Weeks 6–7 | Week 13 |
| 10 Launch + pilot | 10.5 | Week 8, then pilot weeks 9–10 | Week 14, then pilot weeks 15–16 |
| **Total** | **76** | **Gate 1 ≈ week 10** | **Gate 1 ≈ week 16** |

With two engineers, one takes the pipeline (Steps 2, 3, 4.2–4.6 and 9) and the other takes the product (Steps 4.1, 5, 6, 7 and 8). They share Steps 1 and 10. The pipeline track is the critical path, and Gate 0 sits on it in week 6. Giving 2.7 (the CSV upload page) to the product engineer evens out the two tracks. If the pipeline track still runs late, the product engineer can take 3.6 as well.

### Fast track

You can move these to V1 and run the pilot on Inngest's dashboard and scripts instead:

| Deferred work | Days saved |
|---|---|
| The map | 5.5 |
| The source health page | 1 |
| The CSV upload page (import from a script instead) | 1 |
| Confidence calibration (route on raw confidence plus span checks) | 1 |
| Running the performance test at 10,000 leads instead of 100,000 | 0.5 |

That saves 9 days: about a week with two engineers, or two weeks solo. The spec puts the map in Phase 1, but the Phase 1 gate doesn't depend on it.

## Week one

Your track:

- [ ] Pick the launch metro (D1) and confirm one team as the first tenant (D2)
- [ ] Book counsel for the source, privacy-notice and suppression review
- [ ] Ask two or three licensed providers for sample data covering the metro
- [ ] Create the accounts from 0.7, and set a monthly spend limit on the Anthropic key
- [ ] Label the first 50 golden-set items

Engineering:

- [ ] 1.1 Scaffold the app
- [ ] 1.2 CI and preview deploys
- [ ] 1.3 Schema v1 as the first migration
- [ ] 1.7 Sentry and structured logging

## Risks to the plan

| Risk | Effect | Mitigation |
|---|---|---|
| Gate 0 slips (counsel or the vendor contract) | The live source and the pilot slip | Everything up to Step 9 runs on CSV. Business press is a fallback that needs no contract. Hold the week-6 deadline (two engineers) |
| Extraction can't reach 0.95 precision | Agents stop trusting the feed, and the review queue swells | Raise the classifier floor (the spec favors precision over recall). Add hard negatives. Step 3.8 has budget for this |
| Low lead volume in one metro | Fewer than 50 leads during the pilot window | Widen the radius, top up with CSV, backfill 60 days |
| The pilot lands in the holidays | Agents give weak or slow feedback | If the build finishes after December 7, start the pilot in early January |
| V1 features creep into the MVP | The MVP slips | The backlog below is the parking lot. Only gate-critical work goes into the MVP |
| Building solo | The calendar roughly doubles | Use the fast track, or bring in a second engineer for Steps 6–7 |
| Estimation error | ±20% on the total | The 3-day buffer in 10.7. Re-plan at the end of Step 3 |

## Post-MVP backlog

Items are listed in dependency order within each group. Days are rough engineer-day estimates.

### V1: needed for the Phase 2 gate

The spec's Phase 2 gate is "used weekly, with no CSV workaround." These items get there. They total about 40 days: about 5 weeks with two engineers, or 8 solo, in line with the spec's 6–8 weeks for Phase 2.

| ID | Feature | Days | After |
|---|---|---|---|
| V1-1 | Scoring engine (SCO-1 to SCO-7):<br>• seniority dictionary (ENR-4)<br>• proximity to the service area<br>• a pure scoring function with weights set per tenant<br>• per-signal contributions and the top two reasons on each card<br>• Hot, Warm and Cool bands<br>• the unverified cap<br>• `score_audit`<br>• nightly recompute, plus a background rescore when weights change | 8 | The first 200 MVP leads reviewed; D9 |
| V1-2 | Score-driven UI: sort by score by default, with newest, soonest start and nearest as alternatives (FEED-2). Band chips. Map pins colored by band, with a status toggle and legend (MAP-3) | 2 | V1-1 |
| V1-3 | Roles and assignment (SEC-2): a manager role; agents limited to their assigned markets through RLS; assigning and reassigning leads | 4 | — |
| V1-4 | The full filter rail on both feed and map (UI-1) | 3 | V1-1, V1-3 |
| V1-5 | Saved views, shareable within the team (UI-2) | 1.5 | V1-4 |
| V1-6 | Pipeline tab (UI-6, UI-7): a kanban board where dragging a card writes a `lead_event`, with column counts and an "assigned to me" filter | 3 | V1-3 |
| V1-7 | CRM connector framework, with Follow Up Boss as the first connector (INT-2, INT-3, INT-7, PRIV-3):<br>• idempotent push<br>• status webhooks back from the CRM<br>• a stage mapping per tenant<br>• nightly reconciliation<br>• outreach sync<br>• deletion flags | 7 | D8 |
| V1-8 | Daily digest email: new leads in the service area above the user's score threshold, top ten, with a link into the filtered view (INT-4) | 2 | V1-1, V1-5 |
| V1-9 | Bulk select to assign, change status, export or dismiss (FEED-5) | 2 | V1-3 |
| V1-10 | Settings (UI-10): a service-area editor on the map, a weights editor that previews how rankings change, notification preferences | 3 | V1-1 |
| V1-11 | Metrics strip: four numbers with week-over-week change (UI-3) | 1.5 | V1-1 |
| V1-12 | Global search across person, company and title (UI-4, DB-4) | 1.5 | — |
| V1-13 | Pattern library editor (ING-4) | 1.5 | — |

### V1: valuable, not gate-critical

About 24 days in total.

| ID | Feature | Days | After |
|---|---|---|---|
| V1-14 | Second live source (whichever one the MVP didn't use) | 3.5 | Gate 0 for that source |
| V1-15 | Slack delivery of the digest, plus instant alerts for Hot relocations (INT-5) | 1.5 | V1-1 |
| V1-16 | Outbound webhooks, signed and retried (INT-6) | 2 | — |
| V1-17 | Map: draw a radius or polygon to filter, applied to the feed too (MAP-6). Heatmap toggle (MAP-7) | 2.5 | V1-4 |
| V1-18 | Keyboard navigation (FEED-10). The "new leads" pill (FEED-11) | 2 | — |
| V1-19 | Company enrichment and logos: industry, size band, headquarters (ENR-3) | 2 | An enrichment source |
| V1-20 | Estimated compensation band, labeled as an estimate and showing its basis (ENR-6) | 2 | V1-1, V1-19 |
| V1-21 | Lead feedback: agents mark leads good or bad, which feeds evaluation and weight reviews | 1.5 | — |
| V1-22 | Privacy request web form and admin workflow (PRIV-4) | 1.5 | — |
| V1-23 | Phone numbers: column encryption and Do Not Call scrubbing (SEC-5, INT-9) | 2 | A source that supplies phone numbers |
| V1-24 | SAML single sign-on via WorkOS (SEC-1) | 2 | A brokerage that requires it |
| V1-25 | Retention extras: purge leads with no activity for 24 months (DB-10); partition the audit tables by month | 1 | — |

### Later: let the first paying team's usage decide

| ID | Feature | Days |
|---|---|---|
| L-1 | More markets: a market onboarding flow, and search patterns tuned to each market's phrasing | ~5 |
| L-2 | Self-serve SaaS: signup, tenant provisioning, Stripe billing, plan limits | 10–15 |
| L-3 | HubSpot and Salesforce connectors (INT-2) | 4–6 each |
| L-4 | More sources per market: more licensed vendors and press feeds | 3–4 each |
| L-5 | Sales Navigator–assisted capture through a browser extension. The spec rates this medium legal risk, so counsel first | 8–10 |
| L-6 | Move classification and extraction to the Message Batches API (half the price) once LLM spend grows | 2 |
| L-7 | Stronger person and company matching with Splink, or moving the pipeline to Design C's separate data plane once its triggers fire | 5–15 |
| L-8 | Full phone experience (V1 is read-only on phones) | 4–6 |
| L-9 | Compare scores with outcomes (contacted, appointment, closed) and review the weights. Scoring stays a transparent weighted sum, and counsel reviews the changes | 5 |
| L-10 | Fair-housing audit export: rebuild any ranking from its inputs and weights history (SCO-7, AUD-2) | 2 |
| L-11 | AI-drafted outreach copy. The spec defers this until data quality is proven, and it needs an advertising and CAN-SPAM review | 4–6 |
| L-12 | SOC 2 readiness, if you sell to larger brokerages | Ongoing |
| L-13 | A commercial real-estate variant tracking company relocations and headcount growth. This is a separate scope (spec open question 1) | Large |

### Parked by design

These are the spec's deliberate omissions. They're listed so the decision stays visible.

| Item | Why it's parked | Revisit if |
|---|---|---|
| In-app email and calling sequences | The CRM does this better, and it would pull TCPA and CAN-SPAM compliance into the product | Customers turn out not to use a CRM |
| Browsing social connections and viewing profiles | Adds legal exposure for little lead value | — |
| Reporting beyond the four-number strip | Export to a spreadsheet instead | A paying team asks for it repeatedly |
| Sharing leads between tenants, or a lead marketplace | A data-rights problem (PRIV-6) | Only with new data licenses and counsel's approval |
