# Design B — Typed Monolith

> **Thesis:** one TypeScript codebase and one deployable, typed from the database schema to the lead card. The application server owns the logic, Postgres row-level security enforces tenancy beneath it, and Inngest makes the pipeline durable. This is the stack the spec recommends, worked out in detail.

**Status:** selected on Oct 3, 2026. The [workplan](../workplan.md) lays out how to build it.

This is one of three [design options](README.md). Decisions all three share are documented there and not repeated here.

## Stack

| Layer | Pick |
|---|---|
| App and API | Next.js App Router and tRPC on Vercel |
| Database | Neon Postgres with PostGIS and pg_trgm; Drizzle ORM for queries, migrations and RLS policies |
| Background work | Inngest durable functions, served by the same app at `/api/inngest` |
| Auth | Better Auth (magic link, TOTP, organizations), or WorkOS if brokerages require SAML SSO |
| Secrets | Source credentials envelope-encrypted in Postgres under a KMS key; the only secret in the platform's env store is the KMS credential |
| Frontend | First page of cards rendered on the server; client components for the feed (TanStack Query and Virtual) and the map (MapLibre, Supercluster); URL state via `nuqs` |
| Email, Slack | Resend or Postmark; Slack incoming webhooks |

**Variant:** Supabase can host the database instead of Neon. You keep this design's TypeScript-first shape, but use Supabase Auth instead of an auth library. The new-leads pill also gets pushed updates instead of polling.

## Architecture

```mermaid
flowchart LR
  BR["Browser"]
  subgraph vercel["Vercel · one Next.js deployable"]
    APP["UI + tRPC API<br/>tenant-scoped transactions"]
    FN["Pipeline functions<br/>adapters · Claude · enrich · score · deliver"]
  end
  ING["Inngest<br/>cron · throttle · retries · fan-out"]
  DB[("Neon Postgres + PostGIS<br/>RLS")]
  KMS["KMS"]
  EXT["Sources · Claude API · CRM · email · Slack"]

  BR --> APP
  APP --> DB
  APP -- "events" --> ING
  ING -- "invokes steps" --> FN
  FN --> DB
  FN -. "decrypt one credential" .-> KMS
  FN <--> EXT
  EXT -- "CRM webhooks" --> APP
```

## Pipeline

The pipeline is a module inside the app, `src/pipeline`, which Inngest invokes. Each stage is a function triggered by an event, and each external call is a `step.run`. Inngest saves each completed step's result, so when a step fails it retries that step rather than the whole job.

```ts
// Sketch. One fetch per run, so throttling runs is the same as throttling requests.
export const fetchItem = inngest.createFunction(
  {
    id: "fetch-item",
    throttle: { key: "event.data.sourceKey", limit: 60, period: "1m" }, // paces requests per source
    cancelOn: [{ event: "source/disabled", match: "data.sourceKey" }], // SRC-5: disabling drops queued work
    retries: 4,                                                         // exponential backoff between attempts
  },
  { event: "source/item.discovered" },
  async ({ event, step }) => {
    await step.run("take-budget", () => takeBudget(event.data.sourceKey)); // hard per-minute and per-day cap (ING-8)
    const raw = await step.run("fetch", () => adapterFor(event.data).fetch(event.data.item));
    await step.run("store-raw", () => storeRawPost(raw));                    // ON CONFLICT DO NOTHING
  },
);
```

- **Schedule.** An Inngest cron function runs every 15 minutes. It emits `source/run.requested` for each (tenant, source) pair that is due (ING-6).
- **Discover, fetch, store.** Discovery pages through the adapter from its stored cursor and emits one `source/item.discovered` event per candidate.
  - **Budgets** (ING-8). Inngest's throttle paces the fetches. The hard cap is the Postgres counter behind `takeBudget`, and discovery requests count against it too.
  - **Backoff and breaker** (ING-9). The HTTP wrapper backs off on a 429 or 403 and counts consecutive failures in `source_health`. The third failure trips the breaker. That emits `source/disabled` for the pair, which cancels its queued fetches, marks the run `halted` and alerts an admin.
- **Interpret.** A `raw/stored` event runs gate → classify → extract → enrich → score as steps of one function. When volume spikes, `batchEvents` groups classifier calls.
- **Failures.** Each function's `onFailure` handler writes to `dead_letters`. The admin retry button re-sends the original event (ING-14).
- **Re-extraction** (EXT-8) is an admin action. It emits `extract/requested` for every raw post still inside retention, tagged with the new extractor version. A concurrency cap keeps the backfill from crowding out live traffic.
- **Rescoring** (SCO-3, SCO-4) walks a tenant's leads 1,000 at a time and calls the same pure `score()` function the live pipeline uses. It writes a `score_audit` row for each score that changes.

## Tenancy and security

The app connects to Postgres as `app_user`. That role doesn't own the tables and can't bypass RLS (it lacks `BYPASSRLS`). The tables also set `FORCE ROW LEVEL SECURITY` as a backstop. Every tRPC procedure gets its database handle from one middleware, which opens a transaction and sets the tenant context on it:

```ts
export const tenantProcedure = authedProcedure.use(({ ctx, next }) =>
  db.transaction(async (tx) => {
    await tx.execute(sql`select set_config('app.tenant_id', ${ctx.tenantId}, true),
                                set_config('app.user_id',   ${ctx.userId},   true),
                                set_config('app.role',      ${ctx.role},     true)`);
    return next({ ctx: { ...ctx, db: tx } });
  }),
);
```

RLS policies read those settings. A query that forgets `where tenant_id = …` still returns only the tenant's rows.

The weak point is a query that bypasses the middleware altogether. Three guards cover it:

- The root `db` client isn't exported outside `src/server/db`.
- A lint rule bans importing it anywhere else.
- A CI test runs every procedure as tenant A and asserts it never sees tenant B's fixtures.

Use Neon's WebSocket (pooled) driver. Its HTTP driver can't hold the interactive transaction this pattern needs.

- **Admin MFA** (SEC-1). Admin procedures require a session that has completed a TOTP challenge.
- **Protected-field exclusion.** `ScoringInput` is a strict Zod schema with only the allowed fields. A lint rule stops the scoring module from importing the database layer, so it only ever sees what `toScoringInput()` hands it. A unit test fails if anyone adds a field.
- **Audit** (AUD-1). A middleware records view, export, edit and delete events.
- **Export** (SEC-4) is its own permissioned procedure. It logs the row count and filter, re-checks suppression (INT-8), and streams the CSV.
- **Credentials** (SRC-6, SEC-7). Each (tenant, source) credential is a ciphertext row. The fetch step uses KMS to decrypt only the credential it needs. Rotating a credential is a row update.

## Dashboard

- One `buildLeadFilter(filters)` returns a Drizzle `SQL` predicate. `leads.page` and `leads.points` both use it (MAP-6).
- The first 25 cards render on the server and stream with the HTML, which helps meet the 2.5-second interactive target. After that, TanStack Query handles infinite scroll with keyset cursors (FEED-3).
- Filters live in the URL through `nuqs` (MAP-9). Saved views store the same serialized filter (UI-2).
- The new-leads pill polls `leads.newSince({ filters, since })` every 60 seconds. That's a cheap count served from an index (FEED-11).

## Integrations

A status change writes an `outbox` row in the same transaction. An Inngest function delivers outbox rows to the CRM connector, outbound webhooks and Slack.

CRM webhooks arrive at a route handler. It verifies the signature, maps the CRM stage to a lead status, and writes a `lead_event`, using the CRM event id to stay idempotent. A nightly reconciliation catches anything missed (INT-3).

The daily digest is an Inngest cron job per tenant (INT-4). The public privacy-request form posts to its own route handler (PRIV-4).

## Cost and timeline

| Item | Est. $/month |
|---|---|
| Vercel Pro | 20 per seat |
| Neon with PostGIS | 20–70 |
| Inngest | 0–100 |
| Auth (Better Auth runs in the app; WorkOS SSO costs extra per connection) | 0+ |
| Email | 0–20 |
| **Infra total** | **~60–200** |
| LLM ([cost model](README.md#llm-cost-model)) | ~50–65 |

The spec's top-down estimate is 4–6 weeks to the Phase 1 gate. The bottom-up [workplan](../workplan.md) puts it at about 10 weeks for two engineers, including a two-week pilot.

## Risks

| Risk | Mitigation |
|---|---|
| A query escapes the tenant transaction | Non-owner role with FORCE RLS; root client can't be imported; cross-tenant tests in CI |
| An Inngest outage stalls the pipeline | Steps are idempotent and resume on recovery. With a 6-hour target and a 24-hour freshness limit, an hour-long outage costs nothing visible. Inngest can be self-hosted if outages become a pattern |
| Serverless time limits | Every network call is its own step; no step makes more than one fetch or one model call |
| Vendor sprawl | Hold it to four: Vercel, Neon, Inngest, email. Auth runs inside the app |
| Weak person matching in TypeScript | Start with deterministic keys plus trigram similarity. If person-dedup precision misses its target on the golden set, that's the signal to move the pipeline toward Design C |

## Pros

- **It's the spec's own pick.** Its stack table, phase estimates and non-choices apply unchanged, so there's nothing to re-argue.
- **One language, types end to end.** A schema change shows up as a compile error in the card that renders it. Logic is plain TypeScript, which is easy to unit-test and review.
- **Inngest covers most of the pipeline plumbing:** per-source throttling, cron, step-level retries with backoff, cancellation when a source is disabled, failure handlers and run history.
- **Two layers of tenant protection.** Authorization reads as TypeScript middleware, and RLS still catches a mistake.
- **The easiest to hire for and hand off.** Neon can give each pull request its own branch of the database for rehearsing migrations and backfills.

## Cons

- **More vendors.** Vercel, Neon, Inngest, auth and email mean more bills, more dashboards and more failure points. Inngest sits in the path of every pipeline step.
- **Tenant context depends on discipline.** One query on the wrong client skips it. The mitigations work, but they're conventions and tests, not structure.
- **Serverless shapes the code.** Work has to be chopped into short steps. That's mostly healthy, but awkward for long pagination and large exports.
- **TypeScript has thin tooling for data work.** Person matching, company matching and evaluation analysis need more hand-rolled code.
- **More code than A** for the same features: an API layer, auth wiring, and polling instead of push.

**Choose B if:**

- you have TypeScript engineers, or plan to hire them
- you expect to grow toward multi-brokerage SaaS
- you want the default that most teams can maintain
