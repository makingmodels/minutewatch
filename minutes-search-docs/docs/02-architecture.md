# 02 · Architecture

## 1. Shape

One repository deploys as two runtimes that share one Postgres database:

- **Web app** (Next.js on Vercel): pages, server actions, route handlers, Better Auth, search,
  streaming digests. Reads and writes only Postgres. Never calls the Council Gateway API.
- **Pipeline worker**: the same job handlers run in two ways.
  - On Vercel: Cron calls `/api/jobs/drain` every minute; it works queues for up to
    `JOB_DRAIN_MAX_SECONDS` (780 s) and exits.
  - Anywhere else: `pnpm worker` runs the handlers continuously. Used for the first full crawl and locally.

```
Signed-in users ──HTTPS──▶ Next.js web app ──SQL (pooled)──▶ Postgres (Neon)
                                                             ├─ council data + search indexes
Vercel Cron ──▶ /api/jobs/drain ─┐                           ├─ pg-boss job queues
pnpm worker ─────────────────────┴─▶ job handlers ──SQL (direct)─┘└─ users, orgs, saved searches
                                        │
                     ┌──────────────────┼────────────────────┬────────────────┐
                     ▼                  ▼                    ▼                ▼
          Council Gateway API     Claude API           Resend          postcodes.io
          (only data source)      (Batch + streaming)  (email)         (onboarding lookups)
```

## 2. Components

| Component | Package | Responsibility |
| --- | --- | --- |
| Web UI | `apps/web` | Server components for pages; client components only for interactivity |
| Route handlers | `apps/web/app/api` | Search JSON, digest stream, export, onboarding lookups, auth, webhooks, cron drain |
| Data-access layer | `packages/db/dal` | Every query; enforces permissions; returns typed DTOs |
| Schema + migrations | `packages/db` | Drizzle schema, drizzle-kit migrations, raw SQL migrations for extensions and indexes |
| Gateway client | `packages/gateway` | Typed calls, Zod parsing, concurrency and rate limits, retries, circuit breaker, `fetch_log` |
| Pipeline | `packages/pipeline` | Pure functions (clean, score, classify, segment, dedupe) + job handlers |
| Search | `packages/search` | Query parsing, SQL builder, ranker adapter, snippet builder; used by UI and alert matching |
| AI | `packages/ai` | Prompts, Zod output schemas, citation and number verifiers, batch submit/collect, streaming digest, eval |
| Email | `packages/email` | React Email templates, send helper, unsubscribe tokens |
| Queue | pg-boss | Durable jobs, retries with backoff, schedules, singleton keys, dead letters |

## 3. Data flow

1. **Discover**: scheduled jobs find new or changed meetings, decisions and documents upstream.
2. **Fetch**: the gateway client fetches each record and its text; raw is stored with a hash.
3. **Process**: pure stages turn raw into clean, labelled, segmented, de-duplicated data.
4. **Enrich**: AI batch jobs add summaries, topics and signals; OCR fills failed extractions.
5. **Publish**: search vectors written and `stage = 'published'` set in one transaction.
6. **Match**: saved searches run against newly published passages.
7. **Notify**: alerts are sent by email and shown in-app.
8. **Reconcile**: counts and freshness are checked against the API; gaps re-queued.

Details of each stage: `04-data-pipeline.md`.

## 4. Boundaries and rules

- `apps/web` may import `db/dal`, `search`, `ai` (digest only), `email`, `ui`. It may not import
  `gateway` or `pipeline`.
- `pipeline` may import `gateway`, `db`, `ai`, `search` (for matching). It may not import Next.js.
- Pure pipeline functions live in `packages/pipeline/src/pure/*` and do no I/O.
- Two database roles: `app_web` (read council data; write app tables) and `app_worker` (write council
  data). Migrations run as the owner role. See `11-security-privacy.md`.
- Web uses Neon's pooled connection string; worker, migrations and pg-boss use the direct one.

## 5. Scaling notes

- Expected data: ~1.7 M documents, ~17 M passages, ~60 GB total (see `05-data-model.md`).
- Neon autoscaling 1–8 CU in production; raise the floor during the backfill.
- If cron draining falls behind (queue depth growing for 24 h), move the same handlers to a long-running
  container or Vercel Workflows. Handlers must not depend on how they are invoked.
- Phase 2 semantic search adds a `pgvector` column on summaries only; no new service.
