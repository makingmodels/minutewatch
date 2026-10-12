# 13 · Deployment and operations

## 1. Environments

| Env | App | Database | Data | AI |
| --- | --- | --- | --- | --- |
| Local | `pnpm dev` + `pnpm worker` | Docker Postgres 17 (`docker-compose.yml`) | `pnpm db:seed` (3 councils, real pipeline) | Recorded fixtures unless `ANTHROPIC_API_KEY` set |
| Preview (per PR) | Vercel preview | Neon branch of `staging-seed` (3 councils) | Seeded weekly | Fixtures |
| Staging | Vercel (`staging` branch) | Neon branch of production | Full | Live, low budget |
| Production | Vercel Pro (`main`) | Neon, autoscaling 1–8 CU | Full | Live |

**Seed councils** (`pnpm db:seed`): Croydon, North Tyneside, Teignbridge, since 2026-01-01. Configurable:
`pnpm db:seed --councils croydon,haringey --since 2026-01-01`. It runs the real pipeline against the live
API with a lower rate limit (4 rps).

## 2. Configuration

All env vars in `.env.example`, validated at startup with Zod (`packages/config/env.ts`). Separate
schemas for web and worker so each only requires what it uses.

## 3. Vercel

`vercel.json` crons (Pro allows per-minute schedules; verify):

```json
{
  "crons": [
    { "path": "/api/jobs/drain", "schedule": "* * * * *" }
  ]
}
```

- The drain route checks `Authorization: Bearer ${CRON_SECRET}`, sets `export const maxDuration = 800`,
  and works queues until `JOB_DRAIN_MAX_SECONDS`. Scheduling of individual jobs (hourly syncs, nightly
  reconcile, 15-minute alerts) is done by pg-boss `schedule()`, not by extra crons.
- Function region: London (`lhr1`) to sit near users; Neon region: London (`aws-eu-west-2`) or nearest.
- Fluid compute on.

## 4. Neon

- Production branch `main`; `staging` branched from it weekly; preview branches from `staging-seed`.
- Pooled connection for web, direct for worker, migrations and pg-boss.
- Raise compute floor to 4 CU during the full backfill; restore after.
- Point-in-time restore window: plan default; confirm and record.
- BM25: enable `lakebase_text` per Neon docs; then run migration `0002_bm25.sql` and set `SEARCH_RANKER=bm25`.

## 5. CI/CD (GitHub Actions)

On PR: install (pnpm cache) → typecheck → lint → unit → integration (Postgres service) → build →
Vercel preview → migrate preview DB → e2e + axe → `eval:pipeline` and `eval:ai` with fixtures → comment
eval diffs on the PR.

On merge to `main`: run migrations against production (expand-only), promote deployment, smoke test
(`/api/health` checks DB, queue depth, last successful sync), notify.

Destructive migrations: expand → deploy → backfill → contract in a later release.

## 6. First full backfill (runbook)

1. Confirm crawl volume with Poteris; schedule start 20:00 UK.
2. Raise Neon CU floor; set `AI_ENABLED=false` for the crawl.
3. Run `pnpm worker --queues fetch,process --concurrency 8` from a container or laptop with the direct DB URL.
4. Watch `/admin/data-health`: requests/s, error rate, stage funnel.
5. After fetch completes (~24 h) and processing drains: run reconcile; target ≥ 99.5%.
6. Enable AI in priority order (`07` §10) with daily budget; monitor spend.
7. Restore CU floor; record sizes, ratios, durations in `PROGRESS.md`.

## 7. Monitoring and alerts

- Sentry (web + worker), with release tags.
- `/api/health`: DB ok, pg-boss ok, last successful `discover.newest` < 2 h, dead letters = 0.
- Team alerts (email or Slack webhook): freshness lag > 2 h; upstream error rate > 10% for 15 min;
  dead letters > 0; alert send failures; AI spend ≥ 80% of daily budget; reconcile < 99.5%.
- Weekly report email to admins: coverage per council, new councils/committees, quarantine reasons, slowest queries, AI spend.

## 8. Cost (monthly, pilot scale, estimates)

| Item | Estimate |
| --- | --- |
| Neon storage ~60 GB at $0.35/GB-month | ~$21 |
| Neon compute ~2 CU average at $0.106/CU-hour | ~$155 |
| Claude API (new docs, digests, alerts) | ~$105 |
| Resend, Sentry | ~$20 |
| Vercel Pro | per seat + usage |

One-off: ~$570 AI batch + ~$50 extra Neon compute. Check current prices before budgeting.

## 9. Runbooks (create in `docs/runbooks/` during T5.4)

Re-crawl one council · replay a pipeline stage after a rule change (`PIPELINE_VERSION` bump) · upstream
API outage (search keeps working; show "Data last updated" banner) · pause AI jobs · rotate secrets ·
point-in-time restore · handle a "summary is wrong" report · remove a user and their data.
