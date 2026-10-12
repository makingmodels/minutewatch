# 06 · Search

All search runs in our Postgres over published passages. One query builder in `packages/search` serves the
search page, the onboarding preview, the alert preview and alert matching, so results are identical
everywhere.

## 1. Filters type (shared)

```ts
export const SearchFilters = z.object({
  q: z.string().max(300).default(''),
  scope: z.enum(['fulltext', 'summaries']).default('fulltext'),
  councilIds: z.array(z.number().int()).max(50).default([]),
  regions: z.array(z.string()).default([]),
  councilTypes: z.array(z.string()).default([]),
  committeeIds: z.array(z.number().int()).default([]),
  dateFrom: z.string().date().optional(),
  dateTo: z.string().date().optional(),
  datePreset: z.enum(['7d','30d','90d','12m','all','upcoming']).optional(), // rolling; wins over from/to in saved searches
  docTypes: z.array(DocType).default([]),
  topics: z.array(Topic).default([]),
  signals: z.array(Signal).default([]),
  sort: z.enum(['relevance','newest','oldest']).default('relevance'),
  page: z.number().int().min(1).max(50).default(1),
});
```

URL codec with `nuqs`: short keys (`q`, `c`, `r`, `t`, `dt`, `tp`, `sg`, `from`, `to`, `d`, `s`, `p`).

## 2. Query parsing

- Parse with `websearch_to_tsquery('english', q)`: supports `"exact phrase"`, `or`, `-exclude`.
- Trim, collapse whitespace, cap 300 chars. Strip characters that make it error; never surface a syntax error.
- Empty `q` with filters = browse mode: sort by date descending.
- Synonym expansion (small, curated, reviewed): `packages/search/synonyms.ts`, e.g.
  `{ "youth service": ["youth offer", "youth provision"], "closure": ["close", "decommission"] }`.
  Expansions OR'ed in with lower weight. Start with 20 entries from pilot users' topics.

## 3. Execution plan

1. **Resolve councils**: expand two-tier areas: selecting a district also includes its county (users
   think in places), shown as a removable chip "+ Kent County Council".
2. **Candidates**: passages where `tsv @@ query` AND `published_at IS NOT NULL` AND `dup_of IS NULL` AND
   council/date/doc type filters. Topic and signal filters join `documents` on its GIN arrays.
3. **Rank**: through a `Ranker` interface with two implementations, chosen by `SEARCH_RANKER`:
   - `bm25` (Neon): `lakebase_text` BM25 operator against the passages BM25 index.
   - `tsrank` (local, fallback): `ts_rank_cd(tsv, query, 32)`.
   Take the top 2,000 candidates by base score.
4. **Boost** (in SQL over the candidates):
   `score = base × typeBoost × qualityBoost × recency × titleBoost`
   - `typeBoost`: minutes 1.3, decision_notice 1.3, report 1.1, agenda 1.0, agenda_pack 0.9, appendix 0.8, other 0.9
   - `qualityBoost`: good 1.0, poor 0.8
   - `recency`: `0.5 + 0.5 × exp(−age_days / 365)` (relevance sort only)
   - `titleBoost`: 1.25 when `documents.doc_tsv @@ query` (title or summary also matches)
5. **Collapse**: best passage per `coalesce(canonical_id, id)` document; remember the count of other
   matching passages ("3 more matches in this document").
6. **Page**: 20 per page, max 50 pages.
7. **Hydrate** the 20: council, committee, meeting title and date, doc type, summary topline, signals, and
   a snippet.

Use a single SQL statement with CTEs for steps 2–6. Set `statement_timeout = 3000` for the search role.
On timeout: return what the fast path (newest 200 matches by date) gives, with a notice suggesting a filter.

## 4. Snippets

`ts_headline('english', passage_body, query, 'StartSel=<mark>, StopSel=</mark>, MaxFragments=2,
MinWords=12, MaxWords=35, FragmentDelimiter= … ')` on only the passage's slice of `clean` (never the
whole document). Sanitise: escape everything, then restore only `<mark>` and `</mark>`.

## 5. Facets

On the candidate set (before collapse): counts by council (top 20), doc type, topic, signal, year.
Cache facet results for 5 minutes keyed on the normalised filter hash. Counts > 10,000 display "10,000+".

## 6. Summaries scope

`scope = 'summaries'` queries `documents.doc_tsv` (title + summary) instead of passages. Faster and more
"what was this about"; results show the topline instead of a passage snippet.

## 7. Saved-search matching mode

Same builder with `publishedSince: Date` and `publishedUntil: Date`: restricts candidates to
`passages.published_at` within the window and skips facets, boosts and paging. Returns
`(document_id, best_passage_id)` pairs. See `09-saved-searches-and-alerts.md`.

## 8. Quality and performance

- **Benchmark set:** `packages/search/bench/queries.json`: 50 realistic queries (from the product spec
  personas and pilot users) with filters, and for each the IDs of documents judged relevant (graded 0–2).
  Build it in T3.8 with a reviewer.
- **Metric:** nDCG@10 ≥ 0.75 overall; no query below 0.4. Tune boosts and synonyms only against this set
  and record before/after in `PROGRESS.md`.
- **Performance:** run the set on staging (full data) with k6 or autocannon at 10 concurrent users:
  p95 < 800 ms, p99 < 1.5 s, zero errors. Inspect slow plans with `EXPLAIN (ANALYZE, BUFFERS)`.
- **Reserve options** if slow: partition `passages` by year of `doc_date`; materialise per-council
  candidate tables for the 20 busiest councils; raise Neon CU floor.

## 9. Phase 2 (not MVP)

`pgvector` embeddings of document summaries; hybrid ranking by reciprocal rank fusion of BM25 and vector
results. Keep the `Ranker` interface so this drops in.
