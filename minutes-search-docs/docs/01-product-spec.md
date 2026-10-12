# 01 · Product spec

## 1. Purpose

UK councils make decisions that close day centres, cut grants and award contracts, and they record them
in PDF minutes spread across 320 websites. Minutes Search lets the organisations affected find those
decisions early, read them in plain language and get alerted when their issue comes up again.

**One-line promise:** every council minute, searchable, summarised and watched for you.

## 2. Users

| Persona | Role in app | Job to be done | Example |
| --- | --- | --- | --- |
| Policy lead at a charity or VCSE | Member | Spot proposed cuts or closures to services they run or rely on, before the vote | Alert on "refugee support" or "grant reduction" in Haringey |
| Campaign organiser | Member | Find what councils said and decided on an issue, with sources to quote | Every decision on "bus lane" across councils with a climate emergency |
| Local journalist | Member | Scan this week's activity in their area and find a story | All Croydon minutes this week tagged "contracts" |
| Organisation admin | Org admin or owner | Bring in their team, share watchlists | Invite 4 colleagues, share 3 saved searches |
| Platform admin (us) | Platform admin | Approve access, keep the data healthy | Approve requests; check ingestion and data quality |

## 3. Scope

### In the MVP

1. Full mirror of all councils, committees, meetings, decisions, documents and people from the Council
   Gateway API, synced hourly, with reconciliation against API totals.
2. Cleaning and labelling: corrected dates, cancelled flags, document types, agenda-item sections,
   passages, de-duplication, text quality scores, OCR for failed extractions of key document types.
3. AI: cited summary per document; brief per meeting and per decision; topic tags; signals; on-demand
   digest of search results; AI overview in alert emails.
4. Search over the full corpus with filters (council, region, council type, committee, date range,
   document type, topic, signal), sort by relevance or date, two scopes (full text, summaries only).
5. Reader, meeting, decision and council pages, including upcoming meetings.
6. Saved searches: create, rename, share with organisation, follow a colleague's, pause, delete; alerts
   by email and in-app at instant, daily or weekly frequency.
7. Invite-only access, access requests with admin approval, organisations with roles, guided onboarding,
   home briefing.
8. Bookmarks, CSV export (up to 1,000 rows), settings, account export and deletion.
9. Admin console: access requests, users, organisations, ingestion status, data quality, AI coverage and spend.

### Not in the MVP

- Semantic or question-answering search across the corpus (phase 2).
- Attributing statements to individual councillors.
- Spending, supplier or Companies House data.
- Public pages, billing, native mobile apps.

## 4. User stories and acceptance criteria

Each story maps to tasks in `14-build-plan.md`. Acceptance criteria are testable statements; write
Playwright or integration tests for them.

### Access

**A1. Request access.** As a visitor, I can request access so that I can use the tool.
- The request form takes name, work email, organisation and purpose (max 500 characters).
- Submitting shows a confirmation and sends the requester an acknowledgement email.
- Platform admins see the request in `/admin/access` within 1 minute, and receive an email.
- A request from a domain in `AUTO_APPROVE_DOMAINS` is approved automatically and a sign-in link is sent.
- Duplicate requests from the same email within 7 days are merged, not duplicated.

**A2. Sign in.** As an approved user, I can sign in without a password.
- Entering my email sends a magic link and a 6-digit code in the same email; either works once, within 10 minutes.
- An email that is not approved or invited gets the same "check your email" screen (no account
  enumeration) and the email it receives explains how to request access.
- After first sign-in I am offered a passkey; declining does not ask again for 30 days.
- Owners and platform admins must set up two-factor authentication before reaching any other page.

**A3. Invite teammates.** As an org admin, I can invite people by email with a role.
- Invitations expire after 7 days; I can resend or revoke them.
- Accepting an invitation signs the person in and attaches them to my organisation.
- Members cannot invite; admins can invite members; owners can invite admins.

### Onboarding and briefing

**O1. Onboarding.** As a new user, I reach a useful personal briefing in under 3 minutes.
- Six steps as specified in `08-auth-and-onboarding.md`; each can be skipped; progress survives reload.
- Postcode lookup returns every council covering it (district and county for two-tier areas).
- Step 5 shows real results from my councils and topics in under 2 seconds; if the last 30 days are empty
  it widens to 12 months and says so.
- Finishing creates one saved search per chosen issue and lands me on the home briefing.

**O2. Home briefing.** As a returning user, I see what changed since my last visit.
- Shows new matches per saved search since my last view, upcoming meetings in my councils with flagged
  agenda items, and the latest weekly digest.
- Empty state explains why (no saved searches, or nothing new) and offers one next action.

### Search and reading

**S1. Keyword search.** As a member, I can search everything with familiar syntax.
- Supports `"exact phrase"`, `or`, `-exclude`; invalid syntax never errors, it degrades to plain words.
- p95 latency under 800 ms on full production data for the 50-query benchmark (see `12-testing-and-quality.md`).
- Each result shows council, committee, meeting date, document type, signal badges, AI topline and a
  highlighted snippet; duplicates of the same passage are folded.
- Filters and sort are reflected in the URL; reload and share reproduce the same results.

**S2. Filters and facets.** As a member, I can narrow results.
- Filters: councils (multi), region, council type, committee, date range (presets and custom), document
  type, topic, signal, scope (full text or summaries).
- Facet counts show next to each filter value; counts above 10,000 show "10,000+".

**S3. Read a document.** As a member, I can read a cleaned document with its summary.
- The reader shows the agenda-item outline, cleaned text with matches highlighted, next and previous
  match, and the AI summary with numbered citations.
- Clicking a citation scrolls to and highlights the cited passage.
- Documents with unusable text show "Text not available" and a link to the original PDF.

**S4. Summarise results.** As a member, I can get a digest of the top 20 results.
- The digest streams in under 2 seconds to first token, cites results by number and links to them.
- Limited to 20 per user per hour; the limit message says when it resets.

### Saved searches and alerts

**SS1. Save a search.** As a member, I can save any search in one click.
- Dialog pre-fills the name from the query, shows "would have alerted you N times in the last 30 days",
  and offers frequency (instant, daily, weekly, off) and "share with my organisation".
- The saved search appears on `/saved` and on the home briefing immediately.

**SS2. Alerts.** As a member, I am told about new matches.
- New published documents matching a saved search are recorded within 1 hour of publication.
- Instant alerts are sent within 1 hour; daily at my chosen hour (default 07:00 UK); weekly on Mondays.
- A document matching several of my searches appears once per email.
- Every email has one-click pause and unsubscribe that work without signing in.

**SS3. Shared searches.** As a member, I can follow a colleague's shared search.
- Following gives me my own alert settings and my own "new since last view" state.
- Only the owner or an org admin can edit or delete a shared search.

### Admin and data health

**AD1. Data health.** As a platform admin, I can see whether the data is complete and fresh.
- Shows API total vs our total per entity, freshness lag, documents per stage, quarantine counts with
  reasons, dead-letter jobs with retry, AI summary coverage and today's AI spend.

## 5. Success metrics (pilot)

| Measure | Target |
| --- | --- |
| Documents in our database vs API total | ≥ 99.5%, reconciled nightly |
| Lag from API to searchable | < 2 hours (p95) |
| Search latency, p95 full corpus | < 800 ms |
| Summaries judged faithful in a 100-document review | ≥ 95% |
| New user to first briefing | < 3 minutes (median) |
| Pilot users with ≥ 1 saved search after week 1 | ≥ 80% |

## 6. Glossary

| Term | Meaning |
| --- | --- |
| Council Gateway API | Poteris' read-only API over scraped council data. Our only source. |
| Meeting | A committee meeting with a date, documents and sometimes decisions. |
| Decision | A recorded decision (often by a cabinet member or officer), with its own page and documents. |
| Document | One file (usually a PDF) attached to a meeting or decision, with extracted text. |
| Agenda pack / reports pack | One PDF bundling the agenda and all reports for a meeting. Duplicates other documents. |
| Minutes | The formal record of what happened at a meeting. |
| Passage | A ~1,200-character slice of a document's cleaned text: the unit of search. |
| Section | An agenda item within a document, detected by the pipeline. |
| Signal | An AI-extracted flag: cut, closure, consultation, contract, grant, planning decision, key decision coming up. |
| Topic | One of ~30 fixed subject tags (housing, adult social care, refugees and asylum, ...). |
| Published | A document that has passed every pipeline stage and is visible to users. |
| Two-tier area | Places with both a district and a county council; both cover the same postcode. |
