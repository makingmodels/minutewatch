# 09 · Saved searches and alerts

Any search can be saved in one click, shared with the organisation and followed by colleagues, and turned
into alerts with a short AI overview. Matching runs only against newly published passages, so it stays
fast however large the corpus grows.

## 1. Model (see `05-data-model.md`)

- `saved_searches`: the query and filters (`SearchFilters`), owner, optional `organization_id` when
  shared, `query_hash` for fan-out, `last_checked_at`.
- `saved_search_subscriptions`: one row per user following a search (the owner is subscribed on create):
  frequency, channels, `last_viewed_at`, `paused_at`.
- `saved_search_matches`: documents matched, with the best passage.
- `alert_deliveries`: one row per user × channel × frequency × window (unique, for idempotent sends).

Saved filters use **rolling** date presets (`datePreset`), never fixed dates, unless the user sets fixed
dates explicitly.

## 2. Creating

- **From search:** "Save search" button → dialog: name (pre-filled from query and main filter, e.g.
  "Library closures · Croydon"), frequency, "Share with {organisation}" toggle, and a live preview:
  "Would have alerted you 9 times in the last 30 days" (matching mode over the last 30 days).
  Warn above 50 per week: "This will send a lot of alerts. Add a council or signal filter?"
- **From onboarding:** step 6 (`08-auth-and-onboarding.md`).
- **From a council page:** "Watch this council" → saved search with that council and the user's signals.
- **From an agenda item in the reader:** "Alert me about follow-ups" → saved search with the item heading as
  an exact phrase, the council and `datePreset = 'all'` restricted to meetings after today.
- **Following:** on `/saved`, the "Shared with your organisation" tab lists colleagues' shared searches;
  "Follow" creates a subscription.

Limits: 50 saved searches per user; 200 shared per organisation.

## 3. Matching (`match.saved`)

Triggered after each publish batch with `[publishedSince, publishedUntil]`.

1. Load active saved searches (not deleted, at least one unpaused subscription).
2. Group by `query_hash`; run each distinct query once in matching mode (`06-search.md` §7) for the window.
3. Insert `(saved_search_id, document_id, best_passage_id)` into `saved_search_matches` with
   `ON CONFLICT DO NOTHING` (a document matches a search at most once).
4. Advance `last_checked_at` to `publishedUntil` in the same transaction.
5. Matches on documents of meetings in the future are flagged `upcoming` for display ("Coming up: Cabinet, 15 Oct").

Must handle 10,000 saved searches in < 2 minutes per batch; benchmark in T4.4.

## 4. Delivery (`alerts.send`, every 15 minutes)

| Frequency | Window | Sent |
| --- | --- | --- |
| Instant | Since last instant send | Within the hour of publication, batched per user every 15 minutes |
| Daily | Previous 24 h | At the user's `digest_hour` (Europe/London), default 07:00 |
| Weekly | Previous 7 days | Monday at `digest_hour` |

For each due user: collect matches across their subscriptions (by frequency) in the window, de-duplicate
documents across searches (a document listed once, under its first search, with "also matches: …"),
skip if empty (`skipped_empty`), else build and send.

**Email content**
- Subject: `3 new matches: Library closures · Croydon` (single search) or `12 new matches across 4 saved searches`.
- AI overview (2–3 sentences, `07-ai-summaries.md` §8) at the top, if non-empty.
- Per search: name, up to 10 items (council · date · doc type · signal badges · topline · "Read" link to
  the passage), "+ 6 more" link to the search filtered to new matches.
- Footer: "Pause this alert", "Change frequency", "Unsubscribe from all": signed token links to
  `/alerts/manage/[token]` that work without sign-in (token: HMAC of user id + action + expiry 30 days).
- Plain-text alternative; `List-Unsubscribe` and `List-Unsubscribe-Post` headers.
- Sent via Resend; idempotency via the `alert_deliveries` unique key and Resend idempotency keys.

**In-app**
- Home briefing and `/saved` show "N new" per subscription: matches with `matched_at > last_viewed_at`.
  Opening a saved search updates the user's own `last_viewed_at` only.

**Bounces and complaints:** Resend webhook (signature verified) → `email_suppressions`; suppressed users
get in-app only and a banner asking them to check their email address.

## 5. Acceptance tests (from `01-product-spec.md`)

- Publish a fixture document matching a saved search → match row within one `match.saved` run.
- Same document matching three of a user's searches → appears once in the email.
- Daily alert sends exactly once per window even if `alerts.send` runs twice concurrently.
- Pause link works signed out and stops the next send.
- Following a shared search gives independent `last_viewed_at` and frequency.
