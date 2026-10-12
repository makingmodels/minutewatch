# Reference data

Real material captured from the Council Gateway API on 11 Oct 2026. Use it for fixtures, contract tests
and golden sets. Do not edit these files; add new ones alongside.

## `council-gateway-openapi.json`

The upstream OpenAPI 3.1 spec (15 paths). Generate TypeScript types from it in `packages/gateway`
(openapi-typescript). Note that the spec does not describe the quirks in `../03-council-gateway-api.md` §4.

## `api-samples/`

Each file is `{ request, status, body }`. Long `text` fields are truncated to 4,000 characters.

| File | Shows |
| --- | --- |
| `councils_page.json` | Pagination envelope, council fields |
| `meetings_by_date.json` | Date-window listing; note dates vs titles (Q6) |
| `meeting_detail_omits_docs.json` | `/meetings/58403`: compare its `documents` with `documents_by_meeting.json` (Q2) |
| `documents_by_meeting.json` | The correct way to list a meeting's documents |
| `document_with_text.json` | `include_text=true` response |
| `documents_newest_first.json` | Default ordering is newest ID first (used by `discover.newest`) |
| `document_extract_failed.json` | `extract_status: failed`, and a meeting's agenda attached to a decision (Q4, Q10) |
| `document_missing_id_404.json` | A gap in the ID sequence (Q11) |
| `document_file_redirect.json` | `/file` returns 302 to object storage |
| `decisions_page.json`, `decision_detail.json` | Decision fields; empty `purpose`/`content` (Q7) |
| `people_page.json` | Person fields; `name` and `role` quirks (Q7) |
| `search_document_mode.json` | Upstream search hit shape (not used by the product, Q3) |
| `search_classification_mode_empty.json` | Classification mode returns nothing (Q5) |

## `text-samples/`

Real extracted texts (first 25,000 characters; header lines start with `#`). All six are from North
Hertfordshire or North Yorkshire, September 2026. Expand the golden set to ≥ 20 councils in T2.5–T2.8.

| File | Type | Use |
| --- | --- | --- |
| `minutes_1665249.txt` | Minutes | Minutes detection; short procedural minutes |
| `agenda_1665243.txt` | Agenda frontsheet | Agenda detection; **all 24 of its sentences appear verbatim in the reports pack** (de-dupe test) |
| `reports_pack_1665244.txt` | Public reports pack (truncated) | Layout-preserved text, `(Pages 5` … `- 12)` split references, numbered item headings, bundle de-dupe |
| `report_information_note_1665247.txt` | Officer information note (attached to a decision record, Q10) | Report detection; `1.1`-style sub-headings; superscript `th` on its own line |
| `appendix_1665248.txt` | Appendix (`.docx.pdf`) | Appendix detection; URL suffix normalisation; tabular text |
| `table_layout_not_garbled_1665250.txt` | Supplementary licensing information | **Low stop-word ratio but perfectly readable**: must not be scored `garbled` |
