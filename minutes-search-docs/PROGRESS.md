# PROGRESS.md

The build agent updates this file at the end of every task. Humans read it to see where things stand.
Keep entries short. Newest log entries at the top.

## Current status

- **Phase:** 0 (not started)
- **Next task:** T1.1 in `docs/14-build-plan.md`
- **Blocked on:** nothing

## Milestones

Tick a task only when its definition of done in `knowledge.md` §5 is met.

### Phase 1: Foundations
- [ ] T1.1 Monorepo, tooling, CI skeleton
- [ ] T1.2 Local Postgres, Drizzle, first migration
- [ ] T1.3 Council Gateway client with rate limiting and Zod parsing
- [ ] T1.4 Job queue and worker runtime
- [ ] T1.5 Better Auth skeleton (sign-in only, gate all routes)
- [ ] T1.6 Preview deployments on Vercel + Neon branch

### Phase 2: Full ingestion and cleaning
- [ ] T2.1 Reference data sync (councils, committees, ONS enrichment, people)
- [ ] T2.2 Meetings and decisions sync in date windows
- [ ] T2.3 Document discovery (ID walk + newest-first catch-up) and raw fetch
- [ ] T2.4 Validation and normalisation (dates, links, status, quarantine)
- [ ] T2.5 Text cleaning and quality scoring
- [ ] T2.6 Document type classification (rules)
- [ ] T2.7 Agenda-item segmentation and passages
- [ ] T2.8 De-duplication (identical files, bundles, versions)
- [ ] T2.9 Index and publish (search vectors, published flag)
- [ ] T2.10 Reconciliation and admin data-health page
- [ ] T2.11 Full backfill run and coverage report (gate: 99.5% of API documents)

### Phase 3: Search and AI
- [ ] T3.1 Search query builder and ranking adapter (BM25 / ts_rank)
- [ ] T3.2 Search page with filters, facets, result cards
- [ ] T3.3 Document reader, meeting, decision and council pages
- [ ] T3.4 AI document summaries via Batch API, with verification
- [ ] T3.5 OCR of failed extractions
- [ ] T3.6 Meeting and decision briefs
- [ ] T3.7 Results digest (streaming, Sonnet)
- [ ] T3.8 Search quality and performance gate (nDCG, p95 < 800 ms)

### Phase 4: Accounts, onboarding, saved searches
- [ ] T4.1 Organisations, roles, invitations, access requests, admin approval
- [ ] T4.2 Onboarding flow (six steps) and home briefing
- [ ] T4.3 Saved searches (create, share, follow, preview)
- [ ] T4.4 Alert matching and delivery (email + in-app)
- [ ] T4.5 Bookmarks, export, settings, account deletion

### Phase 5: Harden and pilot
- [ ] T5.1 Security review and fixes
- [ ] T5.2 Accessibility audit and fixes
- [ ] T5.3 Load test on staging with full data
- [ ] T5.4 Production deploy, monitoring, runbooks
- [ ] T5.5 Pilot with 3 organisations

## Log

<!-- YYYY-MM-DD · task · one line on what changed · commit hash -->

## Verified facts

Record anything checked against official docs: library versions, plugin names, model IDs, prices,
extension syntax. Include the URL and the date checked.

| Fact | Source | Checked on |
| --- | --- | --- |
| | | |

## Eval scores

| Date | Suite | Before | After | Change made |
| --- | --- | --- | --- | --- |
| | | | | |

## Open questions

Questions for the product owner. Do not block on them; build behind an interface and move on.

- Will Poteris confirm a crawl budget (proposed: 8 concurrent, 10 requests per second, overnight for the
  first crawl) and say whether their search index and classification will be fixed upstream?
- Who are the 3 pilot organisations, and which topics and councils do they need first? (Sets summary
  priority order.)
- Who approves access requests day to day, and which partner email domains are approved automatically?

## Proposals

Ideas outside MVP scope. Not to be built without approval.

## Deviations from spec

Anything built differently from `docs/`, with the reason.
