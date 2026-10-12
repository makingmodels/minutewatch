# 15 · Decisions (settled)

Short architecture decision records. Do not re-open these without a note in `PROGRESS.md` → Proposals.

| # | Decision | Why | Rejected alternatives |
| --- | --- | --- | --- |
| D1 | Mirror the whole Council Gateway API into our Postgres; never query it live for users | Upstream search misses documents (Q3), deep pages are slow (Q1), data needs cleaning (Q5–Q8); users need speed and completeness | Live proxy to `/search`; hybrid local + upstream search |
| D2 | Enumerate documents by ID; meetings/decisions by weekly date windows | IDs are ~98.5% continuous; shallow pages are fast; deep pages take 30–60 s | Paging `/documents` or filtering by council |
| D3 | Link documents via `source_type`/`source_id`, never `/meetings/{id}.documents` | Meeting detail omitted 30% of documents in testing (Q2) | Trusting meeting detail |
| D4 | Store raw immutable; staged, versioned, replayable processing | Cleaning rules will change; re-crawling 1.66 M documents takes a day | Transform on ingest only |
| D5 | Passages (~1,200 chars) as the search unit, with offsets into clean text | Precise snippets and citations; bounded tsvector size; enables bundle de-duplication | Whole-document indexing |
| D6 | Postgres full-text + BM25 (`lakebase_text` on Neon), `ts_rank_cd` fallback, behind a `Ranker` interface | One database, given stack; BM25 quality; local parity. Neon retired `pg_search` (Sept 2026) | Elasticsearch/OpenSearch, Typesense, Meilisearch (extra service to run and sync) |
| D7 | pg-boss in Postgres for jobs; Vercel Cron drains every minute; `pnpm worker` for long runs | No extra infrastructure; durable retries; same handlers everywhere | Redis + BullMQ; Inngest/QStash (possible later) |
| D8 | Claude Haiku 5.5 via Message Batches for bulk AI; Sonnet 5.5 for interactive digests | Bulk cost ~$570 one-off; batch halves price; Sonnet quality where users wait and read | Summarise on demand only (slow first view, no signals for filters/alerts) |
| D9 | Structured JSON summaries with passage citations, verified before storage | Trust: every claim traceable; numbers checked | Free-text summaries |
| D10 | Fixed topic list (~30) and signal list (7) | Filterable, alertable, onboarding chips; stable for evaluation | Free-form tags |
| D11 | Invite-only, passwordless (magic link + code), passkeys, 2FA for owners/admins | Private tool; low friction; strong security | Passwords; open sign-up |
| D12 | One organisation per user in MVP | Simpler UI and access rules | Multi-org switching |
| D13 | Saved searches with separate subscriptions table | Sharing and following with per-user frequency and "new" state | Copying searches per user |
| D14 | Alerts match only newly published passages | Constant cost per batch regardless of corpus size | Re-running full searches each time |
| D15 | OCR with Claude PDF input, priority types only | No OCR binaries to host; cheap; good on layout | Tesseract in a container |
| D16 | Monorepo with strict package boundaries; pure functions for pipeline logic | Testability against real samples; safe agent edits | Single Next.js app folder |
| D17 | UK English, Europe/London display, UTC storage, money in pence | Audience and correctness | — |
