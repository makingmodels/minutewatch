# 03 · Council Gateway API (upstream)

Base URL: `https://councilgateway.poteris.co.uk/council-api` · Read-only · No key required (responses go
through Cloudflare; no published rate limit). OpenAPI 3.1 spec: `reference/council-gateway-openapi.json`.
Real responses: `reference/api-samples/`. Real document texts: `reference/text-samples/`.

**Everything below was measured on 11 Oct 2026.** Re-measure counts during T2.11; do not hard-code them.

## 1. Size

| Entity | Total | Notes |
| --- | --- | --- |
| Councils | 320 | All ModernGov-style (`council_type_label: modern_gov`) |
| Meetings | 132,804 | Late 2023 to 12+ months ahead; 215 dated before Nov 2023 |
| Decisions | 181,356 | Back to 2023 and earlier; at least one dated 2028 (bad data) |
| Documents | 1,661,155 | 82% `source_type: Meeting`, 18% `Decision`; max ID ~1,665,500 |
| People | 19,745 | Councillors with party and role |
| Text | ~38 GB | Mean 22,800 chars per document, median 10,400, p90 51,700, max 520,000 |

Monthly meeting counts: 2024 ~4,350/month; Jul 2026 4,009; Aug 2026 1,068 (summer recess);
Sep 2026 1,956. Confirm with Poteris that recent months are fully scraped.

## 2. Endpoints we use

| Endpoint | Use | Measured speed |
| --- | --- | --- |
| `GET /councils?page&per_page` | Reference sync | ~1 s |
| `GET /councils/{id}/committees` | Reference sync | ~1 s |
| `GET /meetings?council_id&committee_id&date_from&date_to&has_minutes&page&per_page` | Meetings sync in date windows | 0.6–1.9 s for shallow pages |
| `GET /meetings/{id}` | Meeting fields only (see quirk Q2) | ~0.5 s |
| `GET /decisions?council_id&date_from&date_to&is_key&page&per_page` | Decisions sync | ~2 s shallow; 6.4 s at page 1,500 |
| `GET /decisions/{id}` | Decision detail + documents | ~0.4 s |
| `GET /documents?meeting_id&decision_id&council_id&is_minutes&kind&processing_status&page&per_page` | Documents of one meeting or decision; newest-first catch-up | 0.3 s by meeting; 12–60 s by council (avoid) |
| `GET /documents/{id}?include_text=true` | Raw fetch of one document with text | median 0.36 s, p90 0.54 s |
| `GET /documents/{id}/file` | 302 redirect to the PDF in DigitalOcean Spaces (for OCR and "Open PDF") | ~0.3 s |
| `GET /people?council_id&is_councillor&party&page&per_page` | Reference sync | ~0.9 s |
| `GET /healthz` | Circuit-breaker probe | ~0.8 s |

**Do not use** `GET /search` for product search (quirk Q3) or `mode=classification` (returns nothing).

`per_page` maximum is 100. Pagination envelope: `{ items, page, per_page, total }`.

## 3. Record shapes (key fields)

From `components.schemas` in the OpenAPI file; confirm with samples.

- **CouncilOut**: `id`, `name`, `external_id`, `majority_party`, `council_type`, `council_type_label`, `base_scrape_url`
- **MeetingDetail**: `id`, `council_id`, `committee_id`, `name`, `url`, `date` (naive ISO datetime),
  `topline`, `notes`, `agenda`, `about`, `decisions[]`, `documents[]` (incomplete!), `tags[]`, `attendees[]`, `committee`
- **DecisionOut**: `id`, `council_id`, `url`, `decision_maker`, `outcome`, `is_key`, `is_callable_in`,
  `purpose` (usually null), `content` (often empty), `date`, `topline`
- **DocumentDetail**: `id`, `source_type` (`Meeting`|`Decision`), `source_id`, `name` (usually null), `url`,
  `kind` (always `unclassified` today), `extract_status` (`success`|`pending`|`failed`),
  `processing_status` (`waiting`|`processing`|...), `contains_*`/`is_*` flags (all false today),
  `created_at`, `text` (only with `include_text=true`)
- **PersonOut**: `id`, `council_id`, `name`, `role`, `party`, `is_councillor`, `modern_gov_id`, `ocd_id`

## 4. Quirks we must handle

| # | Quirk | Evidence | Handling (see `04-data-pipeline.md`) |
| --- | --- | --- | --- |
| Q1 | Deep list pages are very slow | `/documents` page 8,000: 27 s; page 16,000: 31 s; council filter page 50: 60 s | Enumerate documents by ID; list meetings and decisions in weekly date windows; never page beyond 200 |
| Q2 | `/meetings/{id}` omits documents | 263 of 378 docs returned across 28 meetings; 0 of 31 for Teignbridge; `/documents?meeting_id=` returned all 378 | Link documents via each document's `source_type` + `source_id`. Use `/documents?meeting_id=` when a meeting's list is needed |
| Q3 | Upstream search index is incomplete | Exact phrases from 4 of 20 sampled documents returned zero hits; more were outside the top 10 | Build our own index. Never call `/search` from product code |
| Q4 | Text extraction gaps | 1,200-doc sample: 1,091 success, 98 failed (8%), 11 pending | Re-check pending daily for 30 days; OCR failed key types |
| Q5 | Nothing classified | All `kind: unclassified`, all flags false, `processing_status: waiting`; only 6,885 meetings `has_minutes=true` | Our own classification |
| Q6 | Dates off by one | Meetings titled "Thursday 20 August" dated 19 Aug; "Thursday 1 October" dated 30 Sep; a decision dated 2028-06-27 | Parse date from the title; title wins on disagreement; flag outliers |
| Q7 | Garbage values | Person `name: "803"`; `role: "Councillor, Glenfield Faire"` (ward inside role); empty `purpose`/`content` | Field validators; quarantine; split role and ward |
| Q8 | Bundled duplicates | Reports packs are ~8% of documents but average 131,600 chars (6× others) | Passage-level de-duplication against sibling documents |
| Q9 | URL variants | Mixed `http`/`https`, cache-busting `?T=1`, `.docx.pdf` suffixes | Canonicalise URLs for identity; keep originals |
| Q10 | Cross-linked sources | Meeting documents attached to a decision: `/documents/1665469` (an "Agenda frontsheet") and `/documents/1665244` (a "Public reports pack") both have `source_type: Decision` | Trust `source_type`/`source_id` for linking; record URL-derived meeting reference separately |
| Q11 | Sparse IDs | 197 of 200 random document IDs existed; 404 for the rest | Record 404s as gaps; re-probe gaps weekly in case IDs are back-filled |
| Q12 | Future and cancelled meetings | Meetings listed months ahead; titles containing "CANCEL" | `status` field: scheduled / held / cancelled; recheck scheduled meetings daily |

## 5. Being a good client

- Max 8 requests in flight and 10 requests per second across all workers (token bucket in Postgres or
  pg-boss singleton throttling, so it holds across processes).
- `User-Agent` from `GATEWAY_USER_AGENT` with a contact address.
- Retries: exponential backoff with full jitter (base 1 s, max 60 s, 6 attempts) on network errors,
  429 and 5xx. Respect `Retry-After` if present. 404 is not retried.
- Circuit breaker: 20 consecutive failures pause all gateway jobs for 10 minutes, then probe `/healthz`.
- Log every request to `fetch_log` (endpoint, status, latency, bytes, job id). Keep 30 days.
- Run the first full crawl overnight UK time after agreeing volumes with Poteris.

## 6. Throughput budget

- Full document crawl: ~1.67 M requests at ~20/s (8 in flight) ≈ 23 hours.
- Hourly catch-up: newest-first `/documents` pages until the highest ID we hold, typically 1–5 pages.
- Meetings and decisions windows: ~150 weekly windows × 2 entities, mostly 1–3 pages each, daily.
