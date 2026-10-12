# 05 · Data model

Drizzle ORM, schema in code under `packages/db/src/schema/`. Use `casing: 'snake_case'` in the Drizzle
config so TypeScript names are camelCase and SQL names snake_case. Upstream IDs are primary keys on mirrored
tables, so every ingest is an upsert (`onConflictDoUpdate`).

Extensions and indexes Drizzle cannot express (BM25, trigram) go in hand-written SQL migrations in
`packages/db/migrations/sql/` and run after drizzle-kit migrations.

## 1. Enums

```ts
export const docType = pgEnum('doc_type', ['minutes','draft_minutes','agenda','agenda_pack',
  'supplementary_agenda','report','appendix','decision_notice','equality_assessment','presentation','other']);
export const stage = pgEnum('stage', ['fetched','cleaned','classified','segmented','deduped',
  'summarised','published','quarantined']);
export const textQuality = pgEnum('text_quality', ['good','poor','garbled','empty']);
export const meetingStatus = pgEnum('meeting_status', ['scheduled','held','cancelled']);
export const alertFrequency = pgEnum('alert_frequency', ['instant','daily','weekly','off']);
export const summaryStatus = pgEnum('summary_status', ['complete','partial','failed']);
```

## 2. Council data (mirrored and cleaned)

```ts
export const councils = pgTable('councils', {
  id: integer().primaryKey(),                       // gateway id
  slug: text().notNull().unique(),
  name: text().notNull(),
  gssCode: text().unique(),
  councilType: text(),                              // see 04 §3
  region: text(),
  parentCouncilId: integer(),                       // county for two-tier districts
  majorityParty: text(),
  raw: jsonb().notNull(),
  updatedAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
});

export const committees = pgTable('committees', {
  id: integer().primaryKey(),
  councilId: integer().notNull().references(() => councils.id),
  name: text().notNull(),
  raw: jsonb().notNull(),
}, (t) => [index().on(t.councilId)]);

export const people = pgTable('people', {
  id: integer().primaryKey(),
  councilId: integer().notNull().references(() => councils.id),
  name: text().notNull(),
  role: text(), ward: text(),
  partyRaw: text(), party: text(),                  // canonical party
  isCouncillor: boolean().notNull().default(false),
  raw: jsonb().notNull(),
}, (t) => [index().on(t.councilId)]);

export const meetings = pgTable('meetings', {
  id: integer().primaryKey(),
  councilId: integer().notNull().references(() => councils.id),
  committeeId: integer().references(() => committees.id),
  title: text(),
  url: text(), canonicalUrl: text(),
  startsAt: timestamp({ withTimezone: true }),
  apiDate: timestamp({ withTimezone: true }),
  dateSource: text().notNull().default('api'),      // title | api
  dateFlags: text().array().notNull().default(sql`'{}'`), // date_corrected, date_conflict, weekday_mismatch
  status: meetingStatus().notNull().default('scheduled'),
  briefId: bigint({ mode: 'number' }),              // summaries.id
  rawHash: text().notNull(),
  raw: jsonb().notNull(),
  lastSeenAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
}, (t) => [index().on(t.councilId, t.startsAt), index().on(t.committeeId, t.startsAt), index().on(t.startsAt)]);

export const decisions = pgTable('decisions', {
  id: integer().primaryKey(),
  councilId: integer().notNull().references(() => councils.id),
  decidedOn: date(),
  dateFlags: text().array().notNull().default(sql`'{}'`),
  decisionMaker: text(), outcome: text(),
  isKey: boolean(), isCallableIn: boolean(),
  purpose: text(), content: text(), topline: text(),
  url: text(),
  briefId: bigint({ mode: 'number' }),
  rawHash: text().notNull(),
  raw: jsonb().notNull(),
}, (t) => [index().on(t.councilId, t.decidedOn)]);

export const documents = pgTable('documents', {
  id: integer().primaryKey(),                       // gateway id
  sourceType: text().notNull(),                     // Meeting | Decision
  sourceId: integer().notNull(),
  meetingId: integer().references(() => meetings.id),
  decisionId: integer().references(() => decisions.id),
  councilId: integer().references(() => councils.id),   // null only while orphaned
  docDate: date(),
  title: text(),                                    // derived from URL file name, cleaned
  url: text(), canonicalUrl: text(),
  docType: docType(), docTypeConfidence: real(), docTypeSource: text(), // rules | ai
  upstreamExtract: text(),                          // success | pending | failed
  textSource: text(),                               // api | ocr
  textQuality: textQuality(),
  qualityMetrics: jsonb(),
  rawHash: text(), cleanHash: text(),
  canonicalId: integer(),                           // identical file elsewhere
  supersededBy: integer(),                          // draft → final minutes
  isBundle: boolean().notNull().default(false),
  topics: text().array().notNull().default(sql`'{}'`),
  signals: text().array().notNull().default(sql`'{}'`),
  summaryId: bigint({ mode: 'number' }),
  docTsv: tsvector(),                               // title (A) + summary (B)
  stage: stage().notNull().default('fetched'),
  stageUpdatedAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
  pipelineVersion: integer().notNull(),
  firstSeenAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
  publishedAt: timestamp({ withTimezone: true }),
}, (t) => [
  index().on(t.councilId, t.docDate),
  index().on(t.meetingId), index().on(t.decisionId),
  index('documents_unpublished_idx').on(t.stage).where(sql`stage <> 'published'`),
  index().on(t.publishedAt),
  index().using('gin', t.topics), index().using('gin', t.signals),
  index().using('gin', t.docTsv),
  index().on(t.cleanHash),
]);

export const documentTexts = pgTable('document_texts', {
  documentId: integer().primaryKey().references(() => documents.id, { onDelete: 'cascade' }),
  raw: text(),                                      // exactly as received
  rawOcr: text(),
  clean: text(),
});

export const sections = pgTable('sections', {
  id: bigserial({ mode: 'number' }).primaryKey(),
  documentId: integer().notNull().references(() => documents.id, { onDelete: 'cascade' }),
  ord: integer().notNull(),
  itemNumber: text(), heading: text(), depth: smallint().notNull().default(1),
  start: integer().notNull(), end: integer().notNull(),
}, (t) => [index().on(t.documentId, t.ord)]);

export const passages = pgTable('passages', {
  id: bigserial({ mode: 'number' }).primaryKey(),
  documentId: integer().notNull().references(() => documents.id, { onDelete: 'cascade' }),
  sectionId: bigint({ mode: 'number' }),
  ord: integer().notNull(),
  start: integer().notNull(), end: integer().notNull(),
  hash: text().notNull(),
  dupOf: bigint({ mode: 'number' }),                // excluded from index when set
  // denormalised for filtering without joins
  councilId: integer().notNull(), docDate: date(), docType: docType(),
  tsv: tsvector(),                                  // null when dupOf is set
  publishedAt: timestamp({ withTimezone: true }),
}, (t) => [
  index('passages_tsv_gin').using('gin', t.tsv),
  index().on(t.councilId, t.docDate),
  index().on(t.publishedAt),
  index().on(t.documentId, t.ord),
  index().on(t.hash),
]);

export const summaries = pgTable('summaries', {
  id: bigserial({ mode: 'number' }).primaryKey(),
  subjectType: text().notNull(),                    // document | meeting | decision
  subjectId: integer().notNull(),
  body: jsonb().$type<SummaryJson>().notNull(),     // schema in 07
  status: summaryStatus().notNull(),
  model: text().notNull(), promptVersion: integer().notNull(), inputHash: text().notNull(),
  inputTokens: integer(), outputTokens: integer(), costPence: integer(),
  createdAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
}, (t) => [uniqueIndex().on(t.subjectType, t.subjectId)]);
```

`tsvector` is a Drizzle `customType<{ data: string }>({ dataType: () => 'tsvector' })`.

## 3. Pipeline bookkeeping

| Table | Columns (key ones) | Purpose |
| --- | --- | --- |
| `raw_records` | `entity`, `upstream_id`, `hash`, `body jsonb`, `fetched_at` (PK entity+id) | Raw upstream JSON for meetings, decisions, people, documents |
| `ingest_cursors` | `name` PK, `value text`, `updated_at` | High-water marks (`documents_high_water`, sweep positions) |
| `upstream_gaps` | `entity`, `upstream_id`, `first_seen`, `last_checked`, `status` | 404 IDs to re-probe |
| `fetch_log` | `id`, `endpoint`, `params jsonb`, `status`, `ms`, `bytes`, `job_id`, `at` | Every upstream request; 30-day retention (partition by month or delete job) |
| `quarantine` | `id`, `entity`, `upstream_id`, `stage`, `rule`, `detail jsonb`, `created_at`, `resolved_at` | Invalid records with reasons |
| `pipeline_runs` | `id`, `job`, `started_at`, `finished_at`, `in`, `out`, `quarantined`, `failed`, `notes jsonb` | Stage metrics |
| `reconciliation_snapshots` | `taken_at`, `entity`, `scope jsonb`, `api_count`, `our_count`, `lag_seconds` | Coverage and freshness history |
| `ai_jobs` | `id`, `kind`, `batch_id`, `subject_type`, `subject_id`, `status`, `input_hash`, `submitted_at`, `completed_at`, `error` | Batch tracking |
| `ai_usage` | `day`, `model`, `kind`, `input_tokens`, `output_tokens`, `cost_pence` | Spend tracking and daily cap |

pg-boss creates its own `pgboss` schema.

## 4. App data

Better Auth generates its tables via its CLI (`user`, `session`, `account`, `verification`,
`organization`, `member`, `invitation`, `passkey`, `two_factor`; admin plugin adds `role`, `banned`
fields to `user`). Generate them into `packages/db/src/schema/auth.ts` and commit. Our tables:

```ts
export const profiles = pgTable('profiles', {
  userId: text().primaryKey(),                      // user.id
  jobTitle: text(),
  postcode: text(),
  councilIds: integer().array().notNull().default(sql`'{}'`),
  topics: text().array().notNull().default(sql`'{}'`),
  keywords: text().array().notNull().default(sql`'{}'`),
  signals: text().array().notNull().default(sql`'{}'`),
  onboardingStep: smallint().notNull().default(0),
  onboardedAt: timestamp({ withTimezone: true }),
  digestHour: smallint().notNull().default(7),       // Europe/London local hour
  lastBriefingViewAt: timestamp({ withTimezone: true }),
  passkeyPromptSnoozedUntil: timestamp({ withTimezone: true }),
});

export const accessRequests = pgTable('access_requests', {
  id: serial().primaryKey(),
  email: text().notNull(), name: text().notNull(), organisation: text().notNull(), purpose: text(),
  status: text().notNull().default('pending'),       // pending | approved | declined
  reviewedBy: text(), reviewedAt: timestamp({ withTimezone: true }),
  createdAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
}, (t) => [index().on(t.email), index().on(t.status)]);

export const savedSearches = pgTable('saved_searches', {
  id: serial().primaryKey(),
  ownerId: text().notNull(),
  organizationId: text(),                           // non-null = shared with this organisation
  name: text().notNull(),
  query: text().notNull().default(''),
  filters: jsonb().$type<SearchFilters>().notNull(), // Zod schema shared with search
  queryHash: text().notNull(),                      // normalised query+filters, for fan-out
  createdFrom: text().notNull().default('search'),  // search | onboarding | council | agenda_item
  lastCheckedAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
  createdAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
  deletedAt: timestamp({ withTimezone: true }),
}, (t) => [index().on(t.ownerId), index().on(t.organizationId), index().on(t.queryHash)]);

export const savedSearchSubscriptions = pgTable('saved_search_subscriptions', {
  savedSearchId: integer().notNull().references(() => savedSearches.id, { onDelete: 'cascade' }),
  userId: text().notNull(),
  frequency: alertFrequency().notNull().default('daily'),
  emailEnabled: boolean().notNull().default(true),
  inAppEnabled: boolean().notNull().default(true),
  lastViewedAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
  pausedAt: timestamp({ withTimezone: true }),
}, (t) => [primaryKey({ columns: [t.savedSearchId, t.userId] })]);

export const savedSearchMatches = pgTable('saved_search_matches', {
  savedSearchId: integer().notNull().references(() => savedSearches.id, { onDelete: 'cascade' }),
  documentId: integer().notNull().references(() => documents.id),
  bestPassageId: bigint({ mode: 'number' }),
  matchedAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
}, (t) => [primaryKey({ columns: [t.savedSearchId, t.documentId] }), index().on(t.savedSearchId, t.matchedAt)]);

export const alertDeliveries = pgTable('alert_deliveries', {
  id: bigserial({ mode: 'number' }).primaryKey(),
  userId: text().notNull(),
  channel: text().notNull(),                        // email | in_app
  frequency: alertFrequency().notNull(),
  windowStart: timestamp({ withTimezone: true }).notNull(),
  windowEnd: timestamp({ withTimezone: true }).notNull(),
  items: jsonb().notNull(),                         // [{savedSearchId, documentIds[]}]
  status: text().notNull(),                         // pending | sent | failed | skipped_empty
  providerId: text(), error: text(),
  createdAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
}, (t) => [uniqueIndex().on(t.userId, t.channel, t.frequency, t.windowStart)]); // idempotent sends

export const bookmarks = pgTable('bookmarks', {
  id: serial().primaryKey(),
  userId: text().notNull(),
  documentId: integer().notNull().references(() => documents.id),
  passageId: bigint({ mode: 'number' }),
  note: text(),
  createdAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
}, (t) => [uniqueIndex().on(t.userId, t.documentId, t.passageId)]);

export const auditLog = pgTable('audit_log', {
  id: bigserial({ mode: 'number' }).primaryKey(),
  actorId: text(), action: text().notNull(), targetType: text(), targetId: text(),
  meta: jsonb(), ip: text(),
  at: timestamp({ withTimezone: true }).notNull().defaultNow(),
}, (t) => [index().on(t.actorId, t.at), index().on(t.action, t.at)]);

export const emailSuppressions = pgTable('email_suppressions', {
  email: text().primaryKey(), reason: text().notNull(), at: timestamp({ withTimezone: true }).notNull().defaultNow(),
});
```

## 5. Raw SQL migrations

```sql
-- 0001_extensions.sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;       -- council name autocomplete
CREATE EXTENSION IF NOT EXISTS unaccent;
CREATE INDEX IF NOT EXISTS councils_name_trgm ON councils USING gin (name gin_trgm_ops);

-- 0002_bm25.sql  (Neon only; guarded by SEARCH_RANKER=bm25)
-- Verify exact syntax against Neon's lakebase_text docs before running; record it in PROGRESS.md.
-- CREATE EXTENSION IF NOT EXISTS lakebase_text;
-- CREATE INDEX passages_bm25 ON passages USING lakebase_bm25 (tsv);
```

## 6. Expected size (full corpus)

| Part | Estimate |
| --- | --- |
| `document_texts` raw + clean (TOAST-compressed) | ~25 GB |
| `passages` (~17 M rows) with `tsv` | ~12 GB |
| Search indexes (GIN + BM25) | ~12 GB |
| Everything else | ~5 GB |
| **Plan for** | **~60 GB** |

Measure after the first 50,000 documents of the backfill and record real ratios in `PROGRESS.md`. If
storage runs high, the first lever is not storing `clean` for documents marked `is_bundle` (rebuild on demand).
