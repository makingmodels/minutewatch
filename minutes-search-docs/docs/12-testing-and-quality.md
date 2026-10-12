# 12 · Testing and quality

## 1. Test pyramid

| Layer | Tool | What | Where |
| --- | --- | --- | --- |
| Unit | Vitest | Pure pipeline functions, query parsing, filters codec, verifiers, token signing | `packages/*/src/**/*.test.ts` |
| Integration | Vitest + real Postgres (Docker; testcontainers in CI) | DAL access rules, migrations, job handlers with mocked gateway (fixtures from `docs/reference/api-samples`), search SQL, matching | `packages/*/test/integration` |
| Contract | Vitest | Gateway Zod schemas against recorded fixtures and the OpenAPI file; nightly live smoke test against the real API (read-only, 10 requests) | `packages/gateway/test` |
| End to end | Playwright | User stories in `01-product-spec.md` against a seeded preview | `apps/web/e2e` |
| Accessibility | axe-core in Playwright | Every page in e2e | same |
| Golden sets | custom runners | Pipeline (`eval:pipeline`), AI (`eval:ai`), search relevance (`bench`) | `packages/*/eval` |
| Performance | k6 or autocannon | Search p95, matching throughput | `scripts/perf` |

## 2. Fixtures

- Upstream: `docs/reference/api-samples/*.json` are real responses. Build a fake gateway server (MSW or a
  tiny HTTP server) that serves them plus generated variants (404s, 429s, 500s, slow responses).
- Text: `docs/reference/text-samples/*.txt` are real documents. Expand to the 50-document pipeline golden
  set (§14 of `04-data-pipeline.md`).
- Auth in e2e: a test-only route enabled when `NODE_ENV=test` that issues a session for a seeded user,
  never built into production bundles.

## 3. Required tests (minimum)

- Every acceptance criterion in `01-product-spec.md` has at least one e2e or integration test, named with
  its story ID (`A2: unapproved email gets request-access email`).
- Every quirk Q1–Q12 in `03-council-gateway-api.md` has a test proving it is handled.
- Every DAL function has an "other organisation cannot read/write" test where applicable.
- Cleaning property tests (idempotent, no loss of money sentences) and the table-not-garbled test.
- AI verifier tests: fabricated citation removed; number not in passage removed; > 30% dropped triggers regeneration.
- Alert idempotency under concurrent runs.

## 4. Quality gates (CI blocks merge)

- `pnpm check` green (typecheck, lint with zero warnings, unit + integration).
- e2e green on the preview deployment.
- Golden sets: no metric below its recorded baseline (store baselines in `eval/baseline.json`; update only
  with a note in `PROGRESS.md`).
- Coverage: ≥ 90% lines in `packages/pipeline/src/pure`, `packages/ai/src/verify`, `packages/search`;
  ≥ 70% overall.
- Bundle: search page JS < 150 KB gzipped.

## 5. Performance budgets

| Measure | Budget |
| --- | --- |
| Search p95 / p99 (full data, 10 concurrent) | 800 ms / 1.5 s |
| Document reader server render p95 | 600 ms |
| Onboarding preview | < 2 s |
| Digest time to first token | < 2 s |
| `match.saved` for 10,000 saved searches per batch | < 2 min |
| LCP on mid-range mobile (search, reader, home) | < 2.0 s |
