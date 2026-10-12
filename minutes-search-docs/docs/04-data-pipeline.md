# 04 · Data pipeline: ingestion, cleaning, labelling

The pipeline turns 1.66 million raw upstream records into clean, labelled, de-duplicated, summarised and
indexed documents. It is the foundation of the product: search, summaries and alerts are only as good as
what comes out of here. Build it with more care than anything else.

Read `03-council-gateway-api.md` first for the upstream quirks (Q1–Q12) referred to below.

## 1. Principles

1. **Land raw, then transform.** Store every upstream response and text exactly as received, with a
   SHA-256 hash. All later stages read raw and are deterministic for a given `PIPELINE_VERSION`.
2. **Idempotent.** Every job upserts on upstream IDs and can run twice with the same result.
3. **Staged.** Each document moves `fetched → cleaned → classified → segmented → deduped → summarised →
   published`, or to `quarantined` with a reason. Store `stage`, `stage_updated_at`, `pipeline_version`.
4. **Replayable.** Bumping `PIPELINE_VERSION` re-queues `process.document` for affected documents
   without re-fetching (raw is local). AI stages re-run only if their input hash changed.
5. **Visible only when complete.** Users see `published` documents only.
6. **Quarantine, never drop.** Invalid records go to `quarantine` with stage, rule and detail.
7. **Pure core.** Cleaning, scoring, classification, segmentation and de-dupe logic are pure functions in
   `packages/pipeline/src/pure/` with unit tests on `reference/text-samples/`.
8. **Measured.** Every stage emits counts (in, out, quarantined, failed) and durations to `pipeline_runs`.

## 2. Job graph

```
sync.reference (weekly) ───────────────▶ councils, committees, people
sync.meetings (hourly, windows) ───────▶ meetings ──▶ queues fetch.meeting_documents
sync.decisions (hourly, windows) ──────▶ decisions ─▶ queues fetch.decision_documents
backfill.range (once, 1,670 jobs) ─────▶ queues fetch.document for 1,000 IDs each
discover.newest (hourly) ──────────────▶ queues fetch.document for IDs above high-water mark
recheck.pending (daily) ───────────────▶ queues fetch.document for pending/failed extractions
fetch.document ──(text hash changed)──▶ process.document ──▶ ai.enqueue ──▶ ai.submit ─▶ ai.collect
                                                                                          │
                                         publish ◀───────────────(all stages passed)──────┘
publish (batch) ──▶ match.saved ──▶ alerts.send
reconcile (hourly + nightly)
```

pg-boss settings: `retryLimit: 6`, `retryBackoff: true`, `retryDelay: 5`, `expireInSeconds: 900`,
dead-letter queue per job type. Use `singletonKey` = upstream ID on `fetch.*` and `process.document` so
the same record is never processed twice concurrently.

## 3. Stage 0: reference data (`sync.reference`, weekly)

**Councils.** Upsert all 320 from `/councils`. Enrich from a checked-in lookup
`packages/db/seed/council-lookup.csv` with columns: `gateway_id, name, slug, gss_code, council_type,
region, parent_gss_code`. Build that file once in T2.1:

1. Download the ONS "Local Authority Districts" and "Counties and Unitary Authorities" names-and-codes
   lists for the current year (official ONS Open Geography portal).
2. Match Gateway names to ONS names: normalise (lowercase, strip "council", "borough", "city of",
   "metropolitan", "district", "county", "&"→"and", punctuation), exact match first, then
   Jaro-Winkler ≥ 0.92.
3. Write unmatched names to `council-lookup.unmatched.csv` and resolve them by hand. The file must have
   zero unmatched rows before T2.1 is done.
4. `council_type` ∈ {London borough, metropolitan district, unitary, county, non-metropolitan district,
   Welsh unitary, Scottish council, Northern Irish district, other}. `parent_gss_code` links two-tier districts to their county.

**Committees.** Upsert from `/councils/{id}/committees` (shallow; paginate fully).

**People.** Upsert from `/people` per council. Validation:
- Reject `name` that is empty, purely numeric (`^\d+$`), or under 3 letters → quarantine `person.bad_name`.
- Split `role` of the form `"Councillor, <ward>"` into `role = "Councillor"`, `ward = <ward>`.
- Map `party` through `packages/db/seed/party-map.csv` (raw → canonical) to ~15 canonical parties
  (Conservative, Labour, Liberal Democrat, Green, Reform UK, Plaid Cymru, SNP, Independent, Residents'
  association, Other). Unmapped spellings go to the admin data-quality page; default canonical = "Other".

## 4. Stage 1: discover

### 4.1 Meetings and decisions (`sync.meetings`, `sync.decisions`)

- Iterate **weekly windows** (`date_from`, `date_to` inclusive) from `2023-01-01` to `today + 365 days`.
  Never page beyond page 200 (Q1); if a window would, split it into days.
- Full sweep: daily at 02:00 UK. Incremental: hourly for windows from `today − 14` to `today + 60`.
- Upsert each record (raw JSON stored in `raw_records`). If a meeting is new or its `updated` hash changed,
  queue `fetch.meeting_documents` (calls `/documents?meeting_id=` and queues `fetch.document` per item).
- Decisions: same, with `/decisions/{id}` for documents.
- Scheduled meetings (date in the future or within the last 30 days) are re-listed daily, because agendas,
  supplementary packs and minutes arrive over weeks.

### 4.2 Documents by ID (`backfill.range`, once)

- Read the current maximum ID from page 1 of `/documents` (newest first).
- Queue `backfill.range` jobs for ranges of 1,000 IDs from 1 to max. Each queues `fetch.document` for
  IDs not already held with a current raw hash.
- 404 → insert into `upstream_gaps (entity, id, first_seen, last_checked)`. Re-probe gaps weekly.

### 4.3 Newest documents (`discover.newest`, hourly)

- Read `/documents?page=N&per_page=100` from page 1 (newest first; IDs descending) until a page's
  smallest ID ≤ our high-water mark, max 50 pages. Queue `fetch.document` for unseen IDs.
- Update `ingest_cursors.documents_high_water`.

### 4.4 Re-checks (`recheck.pending`, daily)

- `extract_status = 'pending'` and first seen < 30 days ago → re-fetch.
- `extract_status = 'failed'` → queue for OCR (stage 6) if the type is OCR-eligible.

## 5. Stage 2: fetch (`fetch.document`)

1. `GET /documents/{id}?include_text=true` through the gateway client (rate limits, retries, Zod).
2. Compute `raw_hash = sha256(JSON of metadata fields that matter + text)`.
3. Upsert `documents` metadata and `document_texts.raw`. If `raw_hash` unchanged, stop.
4. Otherwise set `stage = 'fetched'` and queue `process.document`.

## 6. Stage 3: validate and normalise (inside `process.document`)

### 6.1 Linking (Q2, Q10)
- `source_type = 'Meeting'` → `meeting_id = source_id`; `'Decision'` → `decision_id = source_id`.
- If the parent is not in our DB yet: quarantine `document.orphan`; `sync.*` will fetch the parent and
  re-queue. After 72 h orphaned, fetch the parent directly by ID.
- `council_id` comes from the parent (documents have no council field).

### 6.2 Dates (Q6)
Parse meeting titles with this ordered set of patterns (case-insensitive), producing a local date and time:

```
<Committee> - Thursday, 20th August, 2026 7.15 pm
<Committee> - Thursday 20 August 2026 3.00 pm
<Committee> - Monday 9th June, 2025 10.00 am
<Committee> - Wednesday, 30th September, 2026 2.00
Weekday optional; ordinal suffix optional (st|nd|rd|th); comma optional; time optional;
time formats: 7.15 pm | 7:15pm | 19:15 | 10.00
```

Rules:
- If the title yields a date: `starts_at = title date+time (Europe/London)`, `date_source = 'title'`.
  If it differs from the API date by ±1 day, record `date_corrected = true`. If by more than 1 day,
  use the title date but flag `date_conflict` for review.
- If the weekday in the title disagrees with the parsed date, trust the date and flag `weekday_mismatch`.
- No title date: use the API date, `date_source = 'api'`.
- Decisions: use API date; flag dates > 18 months ahead or before 2015 as `date_outlier` and exclude
  them from "upcoming" lists.
- Documents inherit `doc_date` from the parent meeting or decision.

### 6.3 Status
- Title or name contains `cancel` (word boundary, case-insensitive) → `status = 'cancelled'`.
- Future date → `scheduled`; past date → `held`.

### 6.4 URLs (Q9)
`canonical_url = lowercase(scheme+host) + path` with `http→https`, query string removed, `+` and `%20`
treated as spaces, trailing slashes removed. Keep `url` as received.

### 6.5 Schema validation
Every upstream record passes a Zod schema in `packages/gateway/src/schemas.ts`. Unknown extra fields are
allowed (stored in raw). Missing required fields → quarantine `schema.<field>`.

## 7. Stage 4: clean text

Input: `document_texts.raw`. Output: `document_texts.clean`, `text_quality`, `quality_metrics` (jsonb).
Each step is a separate pure function with unit tests. Upstream text is **layout-preserved** (columns of
spaces, right-aligned page references), see `reference/text-samples/`.

1. **Normalise Unicode:** NFKC; convert curly quotes and dashes to plain equivalents for indexing (keep £ and
   other symbols); remove control characters except `\n`; convert `\f` (form feed) to a page-break marker
   `\n<<<PAGE>>>\n`.
2. **Remove page furniture:** split on page markers; any line (trimmed, digits replaced by `#`) that
   appears on ≥ 40% of pages in a document with ≥ 3 pages is furniture (headers, footers, "Page # of #",
   council address blocks). Also remove lines matching `^\s*(Page\s+)?\d+(\s+of\s+\d+)?\s*$`.
3. **Fix superscripts split onto their own line**: a line containing only `st|nd|rd|th` directly above
   or below a line with a day number is merged (sample: `"th\n 16 September 2026"`).
4. **Join hyphenated breaks:** `(\w)-\n\s*(\w)` → `$1$2` only when the joined word is in the dictionary
   (use a bundled English word list) or the fragments are both lowercase.
5. **Unwrap lines into paragraphs:** join a line to the next when the line does not end in sentence
   punctuation, the next starts lowercase or continues a list item, and neither is a heading (stage 9 rules).
   Keep blank lines as paragraph breaks. Collapse runs of spaces to one, except preserve table rows
   (lines with ≥ 2 gaps of ≥ 3 spaces in consecutive lines: keep as a table block, separate cells with ` | `).
6. **Strip page-reference tails:** remove `\(Pages?\s+\d+(\s*-\s*\d+)?\)` and its split forms
   (`(Pages 5` … `- 12)` across lines).
7. **Collapse whitespace** at paragraph ends; trim.

### 7.1 Quality score

Compute on the cleaned text:

| Metric | Definition |
| --- | --- |
| `chars` | Length |
| `alpha_ratio` | Letters ÷ non-space characters |
| `dict_ratio` | Share of tokens (≥ 3 letters) found in the English word list (include a UK local-government term list: "councillor", "s106", "ward", ...) |
| `stop_ratio` | Share of tokens that are common English stop words |
| `avg_token_len` | Mean letters per token |
| `table_ratio` | Share of lines kept as table rows |

Classification:
- `empty`: `chars < 200`
- `garbled`: `dict_ratio < 0.5` **and** `alpha_ratio < 0.6`, or `avg_token_len > 12`
- `poor`: `dict_ratio < 0.7`
- `good`: otherwise

**Do not use `stop_ratio` alone.** `reference/text-samples/table_layout_not_garbled_1665250.txt` has a
low stop-word ratio because it is a table, but it is perfectly readable. It must score `good` or `poor`,
never `garbled`. Make this a unit test.

Effects: `empty` and `garbled` → not segmented for search or summarised; the document is still published
as metadata only with "Text not available". If its type is OCR-eligible, queue OCR.

## 8. Stage 5: classify document type

Types (enum `doc_type`): `minutes`, `draft_minutes`, `agenda`, `agenda_pack`, `supplementary_agenda`,
`report`, `appendix`, `decision_notice`, `equality_assessment`, `presentation`, `other`.

Rules in order; the first rule with a match sets the type and confidence. All string checks on the
URL file name (URL-decoded, lowercase, `+`→space) unless stated.

| # | Rule | Type | Confidence |
| --- | --- | --- | --- |
| 1 | Path has `/documents/g\d+/` and file name starts `printed minutes` | `minutes` | 0.98 |
| 2 | Path `/documents/g\d+/` and file name starts `public reports pack` or contains `reports pack` | `agenda_pack` | 0.98 |
| 3 | Path `/documents/g\d+/` and file name starts `agenda frontsheet` or `agenda` | `agenda` | 0.95 |
| 4 | Path `/documents/b\d+/` and name contains `supplement` | `supplementary_agenda` | 0.95 |
| 5 | Parent is a decision and URL contains `ieDecisionDetails` or name contains `decision notice`, `record of decision` | `decision_notice` | 0.95 |
| 6 | Name contains `minutes` and (`draft` or `unconfirmed`) | `draft_minutes` | 0.9 |
| 7 | Name contains `minutes` | `minutes` | 0.85 |
| 8 | Name matches `^(appendix|annex|annexe|schedule)\b` or contains ` appendix ` | `appendix` | 0.9 |
| 9 | Name contains `eqia`, `equality impact`, `equality analysis` | `equality_assessment` | 0.9 |
| 10 | Name contains `presentation` or ends `.ppt.pdf`/`.pptx.pdf` | `presentation` | 0.85 |
| 11 | Name contains `report` | `report` | 0.8 |
| 12 | Opening 3,000 chars contain `minutes of the meeting` / `present:` / `apologies for absence` + `resolved` | `minutes` | 0.8 |
| 13 | Opening text contains `record of decision` / `decision notice` | `decision_notice` | 0.8 |
| 14 | Opening text contains `notice is hereby given` / `agenda` heading + numbered items + `(Pages` | `agenda` (or `agenda_pack` if chars > 40,000) | 0.7 |
| 15 | No rule matched | `other` | 0.3 |

Below 0.8 confidence, the AI summary call also returns `doc_type` (see `07-ai-summaries.md`) and the higher
confidence wins. Golden set: 200 hand-labelled documents (build in T2.6); target ≥ 95% accuracy on rules
+ AI, ≥ 85% on rules alone.

## 9. Stage 6: segment into sections and passages

### 9.1 Section headings
A line is an agenda-item heading when it matches one of these (after trimming):

```
^(\d{1,2}[A-Za-z]?)[\.\)]\s+([A-Z][A-Z0-9 ,'’&\-–/()]{3,})$           1. APOLOGIES FOR ABSENCE
^(?:Item|Agenda item)\s+(\d{1,2}[A-Za-z]?)[:.\s-]+(.{3,})$             Item 7: Library review
^(\d{1,3}/\d{2,4})\s+(.{3,})$                                           45/26 Budget monitoring (minute numbers)
^(?:Minute|Min\.?)\s*(\d{1,4})[:.\s-]+(.{3,})$                          Minute 12 – Declarations
^(\d{1,2}\.\d{1,2})\s+([A-Z].{3,80})$                                    3.1 Contributions to ... (sub-section; depth 2)
```

Headings in ALL CAPS or Title Case are accepted; sentence-length lines (> 120 chars) are not headings.
Sub-items `1)`, `a)` under a heading are kept as body. Each section stores `item_number`, `heading`,
`depth`, `start`, `end` (character offsets into `clean`).

Documents with no detected headings get one section `heading = null` covering the whole text.

### 9.2 Passages
- Split each section into passages of target 1,200 chars (min 400, max 2,000) on paragraph boundaries;
  fall back to sentence boundaries for long paragraphs; overlap 150 chars with the previous passage.
- A passage never crosses a section boundary.
- Store `start`, `end`, `ord`, `section_id`, normalised `hash` (lowercase, collapse whitespace, strip
  digits used as page numbers) and the search vector (stage 9).
- Passages keep each `tsvector` well below Postgres limits (1 MB per vector; positions capped at 16,383).

## 10. Stage 7: de-duplicate

1. **Identical documents.** Same `clean_hash` on more than one document: the lowest ID is canonical;
   others get `canonical_id` and are not indexed. Their pages still list them.
2. **Bundles.** For each `agenda_pack` / `supplementary_agenda`, compare its passage hashes with the
   passages of sibling documents (same meeting). Passages whose hash already exists in a sibling are
   marked `dup_of = <passage id>` and excluded from the index. If ≥ 90% of a pack is duplicated, mark the
   pack `is_bundle = true`. If a council publishes only the pack, nothing is excluded.
   Use exact hashes first; then near-duplicates with MinHash (5-word shingles, 64 permutations, Jaccard
   ≥ 0.9) within the same meeting only.
3. **Versions.** `draft_minutes` and `minutes` of the same meeting with Jaccard ≥ 0.9: link
   `superseded_by`; search shows the latest.
4. **Cross-meeting reuse.** "Minutes of the previous meeting" inside an agenda pack duplicates the earlier
   meeting's minutes: detect by hash against the previous meeting of the same committee; mark duplicates.

Expected effect: indexed text falls from ~38 GB to ~21 GB. Record the actual ratio in `PROGRESS.md`.

## 11. Stage 8: OCR (failed extraction)

Eligible types (by URL rules from §8): minutes, draft minutes, decision notices, reports. Others wait.
1. Fetch the PDF via `/documents/{id}/file` (follow the redirect). Skip files > 32 MB or > 100 pages;
   mark `ocr_skipped_size`.
2. Send to `claude-haiku-5-5` through the Batch API as a PDF document block with the OCR prompt in
   `07-ai-summaries.md`.
3. Store the result as `document_texts.raw_ocr`, `source = 'ocr'`, then run stages 4–7 on it.

## 12. Stage 9: enrich, index, publish

- AI enrichment per `07-ai-summaries.md` (queued after de-dupe; does not block publishing beyond a
  configurable wait: if the summary is not back within 6 hours, publish without it and attach it later).
- Search vectors per passage:
  `setweight(to_tsvector('english', coalesce(heading,'')), 'A') || setweight(to_tsvector('english', body), 'C')`
  where `body = substring(clean from start+1 for end-start)`.
- Document vector: title (A) + summary topline and key points (B).
- In one transaction: insert passages and vectors, set `stage = 'published'`, `published_at = now()`.
- After each publish batch, queue `match.saved` with the batch's time range.

## 13. Stage 10: reconcile and monitor

| Check | Frequency | Alert when |
| --- | --- | --- |
| Newest API document ID vs newest published ID; time since that ID's `created_at` | Hourly | Lag > 2 hours |
| Totals: API `total` vs our counts for councils, meetings, decisions, documents | Nightly | < 99.5% |
| Per council per month: API meetings vs ours | Nightly | Any council-month off by > 2% |
| Stage funnel: counts per stage; documents stuck in a stage > 24 h | Hourly | > 1,000 stuck |
| Quarantine counts by reason | Nightly | Any reason grows > 10% day on day |
| Dead letters | Continuous | Any |

Results go to `reconciliation_snapshots` and the admin data-health page. Missing IDs are re-queued
automatically.

## 14. Golden sets and tests

- `packages/pipeline/test/golden/` holds 50 documents (raw text + expected clean output excerpts, quality,
  type, section headings, passage count range, duplicate relations). Start with the six files in
  `reference/text-samples/` and add documents from at least 20 different councils.
- `pnpm eval:pipeline` prints per-metric scores: type accuracy, heading precision/recall, quality-class
  accuracy, de-dupe precision/recall. Any drop fails CI.
- Property tests (fast-check) for cleaning: idempotent (`clean(clean(x)) == clean(x)`), never longer than
  input + 1%, never drops sentences containing a £ amount or a number with "%".
