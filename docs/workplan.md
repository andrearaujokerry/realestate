# Workplan — Design B (Typed Monolith)

This plan covers building the MVP of the [Job-Change Lead Engine](<../LinkedIn Job-Change Lead Engine — Requirements.pdf>) on [Design B](designs/design-b-typed-monolith.md), and what comes after it. Ingestion is by [manual paste](designs/README.md#ingestion-manual-paste) (revised Oct 6, 2026). "MVP" here means the spec's Phase 0 and Phase 1. It ends at the spec's Phase 1 gate: **an agent would call 10 of the first 50 leads.**

## At a glance

| | |
|---|---|
| Engineering effort | ~71 engineer-days (realistic range 55–85) |
| Your effort | ~9 days: choosing the market, counsel, golden-set labeling, test pastes, regular pasting, the pilot. Counsel's own time is extra |
| Two engineers | ~7 weeks to build, then a 2-week pilot. Gate 1 around week 9: early-to-mid December 2026 if you started Oct 5 |
| One engineer | ~13 weeks to build, then the pilot. Gate 1 around week 15. With holidays that's late January 2027, or mid-February if the same person also does Step 0 |
| Running cost during MVP | ~$60–200/month infra and ~$15/month LLM. No data licensing |

This is longer than the spec's 5–8 weeks for Phases 0 and 1. Counted bottom-up, the work the spec itself makes MVP scope adds about 25 days that a 4–6 week Phase 1 doesn't have room for:

- compliance machinery
- tenant isolation
- the evaluation harness
- paste segmentation and preview
- launch hardening

A [fast track](#fast-track) cuts about 8.5 days.

**Assumptions behind the estimates:**

- One mid-to-senior TypeScript engineer per track, working five focused days a week.
- Tests and code review are included in each task.
- There's no discount for AI-assisted coding. Building with Claude Code tends to speed up implementation-heavy work: scaffolding, CRUD screens, tests. It doesn't speed up the calendar-bound work: counsel, labeling, pasting, the pilot.

## MVP scope

**In:**

- Manual paste ingestion for one launch market: the paste box, segmenter, preview, paste history with undo, and CSV upload.
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

- automated sources (licensed data, press feeds)
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
| D3 | Is copying posts from LinkedIn by hand OK under its User Agreement? (Gate 0) | Counsel | Week 3 / week 4, before regular pasting starts | If counsel says no, paste press releases and company announcements the same way. Only ingestion changes |
| D4 | Sonnet 5 or 5.5 for extraction | You | Week 3 / week 4 | 5.5. It's the same price, and the golden-set eval will confirm or reject it |
| D5 | Can a short post excerpt outlive the 90-day raw purge? | You + counsel | Week 7 / week 12 | Keep up to 200 characters on the lead if counsel agrees. Otherwise show only the link |
| D6 | Counsel sign-off on the privacy notice, suppression scope and the CRM handoff | Counsel | Before the pilot | — |
| D7 | Pilot agents | You | Week 7 / week 13 | One or two agents who'll say honestly whether they'd call each lead |
| D8 | Which CRM you use | You | Before V1 | Decides the first connector (V1-7) |
| D9 | Seniority as a scoring input | Counsel | Before V1 scoring | See the age-proxy question in the [design notes](designs/README.md#tensions-in-the-requirements-to-settle-now) |

## Steps

Each step lists its tasks in build order. "Days" means engineer-days, and the "After" column lists the tasks that must finish first. Requirement IDs refer to the spec.

### Step 0 — Decide (Phase 0, your track)

This runs alongside engineering. Once the paste box is live, your regular pasting becomes the system's only source of data.

| ID | Task | Your days |
|---|---|---|
| 0.1 | Pick the launch metro and radius. Line up pilot agents | 0.5 |
| 0.2 | Counsel review: copying posts from LinkedIn by hand under its User Agreement, the privacy notice, suppression scope and the CRM handoff → **Gate 0** | 1 + counsel time |
| 0.3 | Build the golden set from posts you copy the way you'll paste them: 200 items, including hard negatives such as promotions, work anniversaries and hiring posts. Keep it in a private spreadsheet until the paste box exists. Label the first 100 by week 3; the review queue adds the rest | 3 |
| 0.4 | Create accounts: GitHub, Vercel, Neon, Inngest, Anthropic (with a spend limit), Resend, MapTiler, Sentry, plus a domain | 0.5 |
| 0.5 | Save 10 raw pastes as segmenter test fixtures, covering every page you'll copy from (feed, post search, single posts). The capture mode in 2.2 saves them | 0.5 |
| 0.6 | Paste regularly once the paste box is live, about 30 minutes a day from week 4, so the pipeline sees real volume before the pilot | 2 |
| 0.7 | Coordinate the pilot (Step 9) | 1.5 |

**Gate 0:** counsel is comfortable with copying by hand, before regular pasting starts.

### Step 1 — Foundation

| ID | Task | Days | After |
|---|---|---|---|
| 1.1 | Scaffold: Next.js App Router, strict TypeScript, tRPC, Drizzle, Tailwind with shadcn/ui, ESLint, Vitest, Playwright | 1 | — |
| 1.2 | CI on GitHub Actions. Each pull request gets a Vercel preview deploy and its own Neon database branch with migrations applied | 1 | 1.1 |
| 1.3 | Schema v1: the spec's core tables and enums, plus the shared additions from the design notes (`lead_sources`, `lead_markets`, `dead_letters`), with paste fields on `source_runs`. Enable PostGIS and pg_trgm. Indexes DB-1 to DB-6 | 2.5 | 1.1 |
| 1.4 | Tenancy (SEC-3): an `app_user` database role that doesn't own the tables and can't bypass RLS; FORCE RLS; the policies; the tenant-scoped tRPC middleware on Neon's pooled driver; a lint ban on importing the root client; a cross-tenant test harness | 2 | 1.3 |
| 1.5 | Auth (SEC-1): Better Auth with magic links sent through Resend, organizations as tenants, admin and agent roles, TOTP required for admins | 2 | 1.4 |
| 1.6 | Observability: Sentry on client, server and Inngest functions; structured logs that carry request and run ids | 0.5 | 1.1 |

**Done when** all three hold:

- A pull request deploys a preview with its own database branch.
- Magic-link login works.
- The cross-tenant test proves tenant A can't read tenant B's data through any procedure.

### Step 2 — Paste intake

This replaces the source adapters. It comes early so you can start pasting real posts while the rest is built.

| ID | Task | Days | After |
|---|---|---|---|
| 2.1 | The `RawPost` Zod schema and the adapter seam, with manual paste as the one adapter (SRC-1, SRC-2, SRC-4) | 0.5 | 1.3 |
| 2.2 | Paste page: a box that captures the clipboard's text and HTML, file drop for CSV and text, and batch hints (location, where it was collected, a posted-around date). A capture mode saves raw pastes as test fixtures | 1 | 1.5 |
| 2.3 | Segmenter rules, in one TypeScript module for browser and server: drop interface text, split on post headers, pull profile and post links from the HTML, resolve relative times. Tested against the fixtures from 0.5 | 2 | 2.2, 0.5 |
| 2.4 | Model fallback: `ingest.segment` sends numbered lines to Haiku 4.5 and gets back each post's line range | 1 | 2.3 |
| 2.5 | Preview table: author, time, first line, captured links, regex-gate hint. Uncheck, merge or split posts before submitting | 1 | 2.3 |
| 2.6 | Submit (ING-7, ING-11, ING-12, PRIV-5): one transaction writes the `source_runs` row and the raw posts. Each post is keyed on its activity id or content hash with ON CONFLICT DO NOTHING, after a suppression check | 1 | 2.1, 1.4 |
| 2.7 | Inngest wiring: typed events, the `/api/inngest` route, one `raw/stored` event per new post after the commit, a per-tenant concurrency cap | 0.5 | 2.6 |
| 2.8 | Paste history (ING-14, ING-15):<br>• each run's counts as its posts move through the pipeline<br>• failed items with retry<br>• undo: delete a run and anything derived from it that nobody has worked yet | 1.5 | 2.7 |
| 2.9 | CSV upload for posts collected in a spreadsheet | 0.5 | 2.6 |

**Done when:**

- Pasting the same page twice produces one set of raw posts.
- A suppressed person is dropped.
- The run's counts show up in paste history.

### Step 3 — Extraction

| ID | Task | Days | After |
|---|---|---|---|
| 3.1 | Regex gate (EXT-1): positive and negative patterns in a `patterns` table, seeded from ING-1 and ING-3. A separate matcher handles LinkedIn's structured "started a new position" updates (ING-2). Logs how much it discards | 1 | 2.6 |
| 3.2 | Classifier on Haiku 4.5 (EXT-2 to EXT-4): structured output with `is_job_change`, a confidence score and the look-alike type. Posts below the confidence floor are dropped and logged | 1.5 | 3.1 |
| 3.3 | Extractor on Sonnet (EXT-5, EXT-6), passing the JSON schema through `output_config.format`. It returns a **list** of people per item, because some posts name several. Each field gets a value, a confidence score and an evidence span (the supporting text). Unstated fields come back null, not guessed | 2 | 3.2 |
| 3.4 | Post-processing (EXT-7): check each evidence span against the text; resolve relative start dates ("starting in March") against the post date; apply the batch location hint when the post names no city; normalize locations; compute overall confidence; route low-confidence records to review | 1.5 | 3.3 |
| 3.5 | Versioning (EXT-8): `extractor_version` is a hash of prompt, schema and model, stored on each lead. A script re-runs extraction over raw posts still inside the retention window | 1 | 3.3 |
| 3.6 | Evaluation harness (EXT-9, EXT-10): a golden-set format; a runner that reports precision, recall and per-field accuracy; a CI job that blocks any prompt, schema or model change that falls below target | 2 | 3.3, 0.3 |
| 3.7 | Calibrate each field's confidence scores against the golden set, stored as versioned config | 1 | 3.6 |
| 3.8 | Iterate on prompts against your real pastes until the targets are met: precision ≥ 0.95, recall ≥ 0.80, field accuracy ≥ 0.90 for title and company and ≥ 0.85 for location and start date | 3 | 3.6 |

**Done when:** the CI eval job meets every EXT-10 target on at least 200 labeled items.

### Step 4 — Enrichment and lead assembly

| ID | Task | Days | After |
|---|---|---|---|
| 4.1 | Gazetteer (ENR-1, ENR-2): load Census place centroids and metro-area (CBSA) definitions into `locations`. Add a normalizer for location strings such as "Greater Austin Area" and "Austin, Texas, United States", a cache, and a geocoder interface with a slot for a fallback provider | 2 | 1.3 |
| 4.2 | Relocation (ENR-5): compare the new and prior metros. Pasted posts carry no profile location, so when the post names no prior city, mark it `unknown` | 0.5 | 4.1, 3.4 |
| 4.3 | Company resolution v1 (ENR-3): normalize the name and fuzzy-match it with trigrams | 0.5 | 1.3 |
| 4.4 | Person and lead dedup (ING-13): match on profile URL first, then on name, company and metro. Keep one lead per person per job change, linked to all its `lead_sources` | 1.5 | 4.3 |
| 4.5 | Market config (metro plus radius, ING-5) and precomputed `lead_markets` rows | 0.5 | 4.1 |
| 4.6 | Lead assembly: create or update the lead with its centroid, precision and provenance, and write a `lead_event` | 1 | 4.2, 4.4, 4.5 |

**Done when:** a pasted post about someone moving from Chicago to Austin becomes one lead. It sits at the Austin centroid, is marked as relocating from Chicago, and carries its provenance chain.

### Step 5 — Review queue

| ID | Task | Days | After |
|---|---|---|---|
| 5.1 | Admin review page (REV-1, REV-2, UI-8): a queue with a backlog badge, the post text beside the extracted fields, and inline edits, including a city picker from the gazetteer for records missing a location. Reviewers can approve, correct and approve, or reject with a reason | 2.5 | 3.4, 4.1, 1.5 |
| 5.2 | Corrections are saved to `extraction_labels`, which grows the golden set. Approved records become leads (REV-3) | 0.5 | 5.1, 4.6 |

**Done when:**

- An admin can clear the queue.
- Every correction shows up as a labeled example in the evaluation harness.

### Step 6 — Feed and detail drawer

| ID | Task | Days | After |
|---|---|---|---|
| 6.1 | App shell: layout, tabs, auth screens, responsive down to tablet width, theme. Quick wireframes for the card and the drawer | 1.5 | 1.5 |
| 6.2 | Lead card with:<br>• name and headline<br>• role and company<br>• location with the Relocating badge<br>• start date, absolute and relative<br>• inline status<br>• post excerpt, with a link when the paste captured one<br>• actions: mark contacted, add note, dismiss<br>• Unverified marker<br>Company logos come later (V1-19) | 2 | 6.1 |
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
| 8.3 | Opt-out without deletion. A privacy notice page with an address for requests (PRIV-4, PRIV-5) | 0.5 | 2.6 |
| 8.4 | Audit middleware recording views, exports, edits and deletes, plus an admin audit viewer (AUD-1) | 1 | 1.4 |
| 8.5 | Nightly purge of raw posts older than 90 days. It runs after provenance has been copied to the lead, along with the excerpt if D5 allows it (DB-7) | 0.5 | 4.6 |
| 8.6 | Paste settings: admins only, or agents too | 0.5 | 2.6 |

**Done when:**

- Deleting a test person removes every trace of them.
- Pasting that person again is blocked by the suppression list.

### Step 9 — Launch and pilot

| ID | Task | Days | After |
|---|---|---|---|
| 9.1 | Production setup: environment, domain, Resend SPF and DKIM, Neon point-in-time restore, production keys for Inngest, Sentry alerts | 1 | Steps 1–8 |
| 9.2 | Playwright smoke tests: login, paste and preview, feed, drawer, status change, export, review approval, deleting a person | 1.5 | Steps 2, 5, 6, 8 |
| 9.3 | Performance test on 100,000 synthetic leads: feed p95 under 500 ms, map p95 under 800 ms, 5,000 pins drawn in under 1 second | 1 | Step 7 |
| 9.4 | Security and accessibility pass: cross-tenant tests, dependency audit, secret scan, security headers. Keyboard focus, contrast and labels on the feed | 1.5 | Steps 1–8 |
| 9.5 | Runbooks for handling a deletion request, undoing a bad paste, and re-running extraction | 0.5 | Step 8 |
| 9.6 | Pilot support: two weeks with one or two agents. Review the first 50 leads with them, and sort what they miss into fixes or the backlog | 2 | 9.1 |
| 9.7 | Buffer for bug fixes | 3 | — |

**Gate 1 (MVP done)** when all of these hold:

- The pilot agent would call at least 10 of the first 50 leads (the spec's gate).
- Golden-set metrics meet the EXT-10 targets on at least 200 items.
- Two weeks of regular pastes processed without manual fixes.
- Suppression, hard delete and export logging pass their tests.
- Feed and map meet their p95 targets at 100,000 leads.
- Counsel has signed off on copying by hand and the privacy notice.

## Effort and schedule

| Step | Days | Two engineers | Solo |
|---|---|---|---|
| 0 Decide (your track) | 9 (yours) | Weeks 1–2; pasting from week 4 | Weeks 1–2; pasting from week 5 |
| 1 Foundation | 9 | Weeks 1–2 | Weeks 1–2 |
| 2 Paste intake | 9 | Weeks 2–3 | Weeks 2–4 |
| 3 Extraction | 13 | Weeks 3–5, tuning in week 7 | Weeks 4–7 |
| 4 Enrichment | 6 | Weeks 5–6 | Weeks 7–8 |
| 5 Review queue | 3 | Week 6 | Week 8 |
| 6 Feed + drawer | 10 | Weeks 3–5 | Weeks 9–10 |
| 7 Map | 5.5 | Weeks 6–7 | Week 11 |
| 8 Compliance + admin | 5 | Week 7 | Week 12 |
| 9 Launch + pilot | 10.5 | Week 7, then pilot weeks 8–9 | Week 13, then pilot weeks 14–15 |
| **Total** | **71** | **Gate 1 ≈ week 9** | **Gate 1 ≈ week 15** |

With two engineers, one takes the pipeline and the other takes the product. They share Steps 1 and 9.

- **Pipeline:** 2.1, 2.3, 2.4, 2.6–2.9, Step 3 and Step 4.
- **Product:** 2.2, 2.5, and Steps 5–8.

With no outside data source to wait for, the critical path runs through paste intake, extraction and enrichment, then launch. The two tracks come out within about a day of each other.

### Fast track

You can move these to V1:

| Deferred work | Days saved |
|---|---|
| The map | 5.5 |
| The segmenter's model fallback (fix mis-splits by hand in the preview) | 1 |
| Confidence calibration (route on raw confidence plus span checks) | 1 |
| CSV upload (paste only) | 0.5 |
| Running the performance test at 10,000 leads instead of 100,000 | 0.5 |

That saves 8.5 days: about a week with two engineers, or two weeks solo. The spec puts the map in Phase 1, but the Phase 1 gate doesn't depend on it.

## Week one

Your track:

- [ ] Pick the launch metro (D1) and confirm one team as the first tenant (D2)
- [ ] Book counsel, with copying from LinkedIn by hand first on the list
- [ ] Create the accounts from 0.4, and set a monthly spend limit on the Anthropic key
- [ ] Copy the first 50 golden-set posts into a private spreadsheet, the same way you'll paste them

Engineering:

- [ ] 1.1 Scaffold the app
- [ ] 1.2 CI and preview deploys
- [ ] 1.3 Schema v1 as the first migration
- [ ] 1.6 Sentry and structured logging

## Risks to the plan

| Risk | Effect | Mitigation |
|---|---|---|
| Counsel advises against copying from LinkedIn at volume | The ingestion plan changes | Paste press releases and company announcements instead, or add a licensed source later (L-4). Only the ingestion seam changes |
| Extraction can't reach 0.95 precision | Agents stop trusting the feed, and the review queue swells | Raise the classifier floor (the spec favors precision over recall). Add hard negatives. Step 3.8 has budget for this |
| Not enough pasting | Fewer than 50 leads by the pilot | Paste about 100 posts a week from week 4. Widen the radius |
| LinkedIn changes its page layout | The segmenter's rules stop splitting cleanly | The Haiku fallback, the preview and the fixture tests. Capture fresh fixtures when it happens |
| Missing locations swamp the review queue | Reviewing takes longer than pasting | Paste from location-focused searches with a batch hint. Use the city picker in 5.1 |
| The pilot lands in the holidays | Agents give weak or slow feedback | If the pilot would overlap late December, start it in early January |
| V1 features creep into the MVP | The MVP slips | The backlog below is the parking lot. Only gate-critical work goes into the MVP |
| Building solo | The calendar roughly doubles | Use the fast track, or bring in a second engineer for Steps 6–7 |
| Estimation error | ±20% on the total | The 3-day buffer in 9.7. Re-plan at the end of Step 3 |

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

About 21 days in total.

| ID | Feature | Days | After |
|---|---|---|---|
| V1-14 | Duplicate check in the preview: flag posts already in the engine before you submit | 1 | — |
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
| L-4 | Automated sources: a licensed provider or business-press feeds. This brings back the machinery removed for manual paste: scheduling, request budgets, backoff and the breaker, robots.txt, source credentials, source health. Counsel first | 8–10 for the first |
| L-5 | Sales Navigator–assisted capture through a browser extension. The spec rates this medium legal risk, so counsel first | 8–10 |
| L-6 | Run re-extraction backfills through the Message Batches API (half price; nobody waits on a backfill) | 1 |
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
