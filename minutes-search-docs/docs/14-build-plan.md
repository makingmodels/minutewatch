# 14 · Build plan

Work top to bottom. One task at a time. A task is done only when its **Done when** items are demonstrated
and the definition of done in `knowledge.md` §5 is met. Task IDs match `PROGRESS.md`.

Phases 2 and 3 overlap: start AI summaries (T3.4) for the most recent year as soon as T2.9 publishes
documents; the older backfill continues in parallel.

---

## Phase 1: Foundations (week 1)

**T1.1 Monorepo, tooling, CI skeleton**
- Build: pnpm workspace + Turborepo with the layout in `knowledge.md` §4; TypeScript strict; ESLint
  (zero warnings) + Prettier; Vitest; root scripts from `knowledge.md` §5 (stubs allowed for later ones);
  GitHub Actions running `pnpm check`; `packages/config/env.ts` validating `.env.example` variables.
- Done when: `pnpm check` passes on a clean clone; CI green; missing env var fails startup with a clear message.

**T1.2 Local Postgres, Drizzle, first migration**
- Build: `docker-compose.yml` (Postgres 17); Drizzle config with `casing: 'snake_case'`; council-data and
  pipeline tables from `05-data-model.md`; raw SQL migration `0001_extensions.sql`; DB roles from `11` §2.
- Done when: `pnpm db:up && pnpm db:migrate` on an empty DB creates everything; integration test inserts
  and reads a council, meeting and document.

**T1.3 Council Gateway client**
- Build: `packages/gateway` with types generated from `docs/reference/council-gateway-openapi.json`
  (openapi-typescript), Zod schemas for every response used, token-bucket rate limit (cross-process via
  Postgres), concurrency cap, retries with full jitter, circuit breaker, `fetch_log` writes, User-Agent.
- Done when: unit tests cover retry on 429/5xx, no retry on 404, breaker opens after 20 failures; contract
  tests parse every file in `docs/reference/api-samples/`; a live smoke script fetches 5 endpoints.

**T1.4 Job queue and worker runtime**
- Build: pg-boss setup (queues created explicitly, dead-letter queues, retry policy from `04` §2);
  `pnpm worker` entry; `/api/jobs/drain` route with `CRON_SECRET` and time budget; job registry shared by both.
- Done when: an integration test enqueues 100 no-op jobs and both runtimes drain them; a failing job lands
  in dead letters after retries; drain route returns 401 without the secret.

**T1.5 Better Auth skeleton**
- Build: Better Auth with Drizzle adapter and magic link + email OTP; closed sign-up hook; middleware gate;
  `requireSession()`; console email transport locally; sign-in and verify pages per `10-ux-spec.md`.
- Done when: e2e: allowed email signs in via link and via code; unknown email gets the same screen and a
  request-access email; every non-public route redirects to sign-in when signed out.

**T1.6 Preview deployments**
- Build: Vercel project, Neon project with `staging-seed` branch, preview branches per PR, migrations in CI,
  Sentry wired.
- Done when: a PR produces a working preview URL with its own DB branch and green e2e.

## Phase 2: Full ingestion and cleaning (weeks 2–3)

**T2.1 Reference data sync**
- Build: `sync.reference`; `council-lookup.csv` per `04` §3 with zero unmatched; committees; people with
  validation and `party-map.csv`.
- Done when: all 320 councils have GSS code, type, region; districts link to counties; people with
  numeric names are quarantined; test on `api-samples/people_page.json`.

**T2.2 Meetings and decisions sync**
- Build: weekly-window sweeps with day-splitting; hourly incremental; title date parsing and flags (`04` §6.2–6.3); raw records.
- Done when: unit tests cover every title format in `04` §6.2 and the 19-vs-20 Aug case; for the 3 seed
  councils our meeting counts match the API per month exactly.

**T2.3 Document discovery and raw fetch**
- Build: `backfill.range`, `discover.newest`, `fetch.document`, `upstream_gaps`, `recheck.pending`;
  linking via `source_type`/`source_id` with orphan handling.
- Done when: for the seed councils, every document returned by `/documents?meeting_id=` for every meeting
  is in our DB (test Q2: count must exceed what `/meetings/{id}` lists); re-running fetch on unchanged
  docs does no processing (hash check).

**T2.4 Validation and normalisation**
- Build: Zod validation, quarantine with reasons, URL canonicalisation, status, date inheritance.
- Done when: quarantine page lists reasons; fixture with missing fields is quarantined, not dropped.

**T2.5 Text cleaning and quality scoring**
- Build: pure functions per `04` §7; quality metrics and classes.
- Done when: unit + property tests pass; all six `reference/text-samples` produce expected outputs
  (write snapshot expectations by hand, reviewed); table sample is not `garbled`; `(Pages 5 … - 12)` split
  references removed; `th` superscript merged.

**T2.6 Document type classification (rules)**
- Build: rules table `04` §8; 200-document labelled golden set from ≥ 30 councils.
- Done when: rules-only accuracy ≥ 85% on the golden set; `pnpm eval:pipeline` reports it.

**T2.7 Segmentation and passages**
- Build: heading detection and passage splitter per `04` §9.
- Done when: the reports-pack sample yields sections 1–7 with correct headings; passages respect min/max
  and section boundaries; heading precision ≥ 90%, recall ≥ 85% on the golden set.

**T2.8 De-duplication**
- Build: identical-file, bundle, version and previous-minutes detection per `04` §10.
- Done when: every passage of `agenda_1665243.txt` is found inside `reports_pack_1665244.txt` (same
  meeting; all 24 of its sentences appear verbatim in the pack), so the pack's matching passages are marked
  `dup_of`; de-dupe precision ≥ 95% on golden pairs; measured ratio recorded.

**T2.9 Index and publish**
- Build: weighted search vectors, document vectors, transactional publish, `match.saved` trigger hook.
- Done when: a published document is searchable via a raw SQL `@@` query; unpublished documents never are.

**T2.10 Reconciliation and data-health page**
- Build: `reconcile` job, snapshots, `/admin/data-health` per `04` §13 and `10` §4.
- Done when: page shows totals vs API, lag, stage funnel, quarantine, dead letters for the seed data.

**T2.11 Full backfill and coverage report**
- Build: run the runbook in `13` §6 on staging.
- Done when: ≥ 99.5% of API documents present; lag < 2 h for 48 hours; sizes, ratios, durations and real
  quality/type distributions recorded in `PROGRESS.md`. **Gate for Phase 3 completion.**

## Phase 3: Search and AI (weeks 3–4)

**T3.1 Search query builder and ranker adapter**
- Build: `packages/search` per `06`; `tsrank` and `bm25` rankers behind `SEARCH_RANKER`; matching mode.
- Done when: unit tests for parsing edge cases (unbalanced quotes, only exclusions, 300+ chars); integration
  test returns expected order on a small fixture corpus; both rankers pass the same tests where available.

**T3.2 Search page**
- Build: UI per `10` §4 with URL state, facets, result cards, folding, empty states, export.
- Done when: e2e for S1 and S2; axe clean; JS budget met.

**T3.3 Reader, meeting, decision, council pages**
- Done when: e2e for S3 (citations scroll to passages once T3.4 lands; outline and match navigation now);
  unusable-text state shown for an `empty` document.

**T3.4 Document summaries**
- Build: `packages/ai` batch submit/collect, prompt v1, schema, verifiers, budget guard, priority queue;
  100-document AI golden set with references.
- Done when: `eval:ai` meets the launch gate in `07` §11; verifier unit tests pass; cost per 1,000
  documents measured and recorded.

**T3.5 OCR**
- Done when: 20 failed-extraction fixtures produce text that passes quality `good`/`poor` in ≥ 80% of cases.

**T3.6 Meeting and decision briefs**
- Done when: briefs exist for all meetings of the seed councils with summarised documents; upcoming meetings use "will consider".

**T3.7 Results digest**
- Done when: e2e S4; first token < 2 s on staging; rate limit enforced with reset time.

**T3.8 Search quality and performance gate**
- Build: 50-query benchmark with judged relevance; perf script.
- Done when: nDCG@10 ≥ 0.75 (none < 0.4); p95 < 800 ms on staging full data. Record results.

## Phase 4: Accounts, onboarding, saved searches (week 5)

**T4.1 Organisations, roles, invitations, access requests**
- Done when: e2e A1 and A3; admin approves into new and existing org; 2FA enforced for owners/admins; IDOR tests.

**T4.2 Onboarding and home briefing**
- Build: six steps per `08` §5, postcode lookup, preview endpoint, home briefing per `10` §4.
- Done when: e2e O1 (under 3 minutes scripted, preview < 2 s), O2, empty states.

**T4.3 Saved searches**
- Done when: e2e SS1 and SS3; preview count matches matching-mode result over 30 days.

**T4.4 Alert matching and delivery**
- Done when: e2e/integration SS2 including idempotency and de-duplication across searches; signed
  pause/unsubscribe works signed out; 10,000-search matching benchmark < 2 min.

**T4.5 Bookmarks, export, settings, account deletion**
- Done when: export CSV ≤ 1,000 rows with correct columns; delete account removes data per `08` §6; download-my-data JSON complete.

## Phase 5: Harden and pilot (week 6)

**T5.1 Security review**: `11` §7 checklist complete; findings fixed or logged with rationale.
**T5.2 Accessibility audit**: axe clean; manual screen-reader pass on search, reader, onboarding.
**T5.3 Load test**: budgets in `12` §5 met on staging.
**T5.4 Production deploy**: runbooks written; monitoring and alerts firing in a test; health endpoint green.
**T5.5 Pilot**: 3 organisations onboarded; feedback logged; success metrics in `01` §5 measured after week 1.
