# 08 · Auth, access and onboarding

The tool is private. Nobody gets in without an invitation or an approved request, and every new user
leaves onboarding with a personal briefing built from real data and alerts already set up.

## 1. Better Auth setup

- Drizzle adapter on our Postgres; tables generated with the Better Auth CLI into `packages/db`.
- Plugins: **magicLink**, **emailOTP**, **organization** (members, invitations, roles), **admin**
  (platform roles, ban, impersonate for support), **passkey**, **twoFactor** (TOTP + backup codes).
  Verify current plugin names and options in the Better Auth docs; record versions in `PROGRESS.md`.
- Optional Google social sign-in, enabled only when `GOOGLE_CLIENT_ID` is set.
- Sessions: 30-day rolling, cookie `Secure`, `HttpOnly`, `SameSite=Lax`; session refresh on use.
- Sign-up is **closed**: Better Auth must not create users on first sign-in unless the email is
  (a) invited, (b) approved in `access_requests`, (c) on `AUTO_APPROVE_DOMAINS`, or (d) in
  `PLATFORM_ADMIN_EMAILS`. Enforce in a Better Auth `before` hook on user creation; reject otherwise.

## 2. Roles

| Role | Scope | Can |
| --- | --- | --- |
| Member | Organisation | Search, read, save searches, share to own org, bookmarks, export |
| Admin | Organisation | Member + invite members, revoke invitations, edit/delete org shared searches, remove members |
| Owner | Organisation | Admin + invite admins, transfer ownership, delete organisation |
| Platform admin | Whole app | Approve access requests, create organisations, manage users, see admin console, retry jobs |

Every user belongs to exactly one organisation in the MVP (simpler UI; multi-org later). Two-factor is
required for owners and platform admins.

## 3. Public routes (no session)

`/sign-in`, `/sign-in/verify`, `/request-access`, `/invite/[token]`, `/api/auth/*`, `/api/webhooks/*`,
`/api/jobs/drain` (secret-protected), `/alerts/manage/[token]` (signed one-click pause/unsubscribe),
`/legal/privacy`, `/legal/terms`. **Everything else requires a session.** Middleware redirects to
`/sign-in?next=…`; server code re-checks via the DAL.

## 4. Ways in

```
Invitation email ──▶ /invite/[token] ──▶ sign in (email prefilled) ──▶ onboarding
Request access ──▶ admin approves (or auto-approve domain) ──▶ approval email with sign-in link ──▶ onboarding
Platform admin creates org + owner invite ──▶ owner signs in ──▶ onboarding (+ "invite your team" step)
```

- **Request access form:** name, work email, organisation, "What will you use this for?" (≤ 500 chars).
  Show: "Thanks. We review requests within 2 working days. We've emailed you a copy."
  Platform admins approve into an existing organisation or create a new one; declining sends a polite email.
- **Sign-in screen:** one field (email) and one button "Email me a sign-in link". The email contains a
  magic link **and** a 6-digit code ("or enter this code"), both valid 10 minutes, single use. Same
  "Check your email" screen for every address (no enumeration); unapproved addresses receive a
  "request access" email instead of a link.
- **After first sign-in:** offer a passkey ("Sign in faster next time"); skip snoozes for 30 days.

## 5. Onboarding (six steps)

Route group `(onboarding)/welcome/[step]`. Progress bar, Back, Skip, Continue. Each step saves to
`profiles` on Continue, so reloads resume. Users who skip everything get sensible defaults (their org's
area if known, no topics) and land on the home briefing with a prompt to set up alerts.

| Step | Screen | Behaviour |
| --- | --- | --- |
| 1. Welcome | "Welcome to Minutes Search, {first name}." Name, job title (optional), organisation shown | Organisation pre-filled from invitation; editable name |
| 2. Your patch | "Which area do you work in?" Postcode or place input; or search councils by name; region chips | Postcode → `api.postcodes.io/postcodes/{pc}` → admin district and county GSS codes → our councils. Show "3 councils cover SE25: Croydon (London borough) …" pre-ticked. Place names via postcodes.io places endpoint. Multiple areas allowed |
| 3. Your issues | "What do you care about?" Topic chips (from the fixed list) + free keywords ("day centre", "Sanctuary") | At least one topic or keyword to continue (or Skip) |
| 4. What to watch for | Signal toggles: cuts and closures, consultations, contracts and procurement, grants, planning decisions, key decisions coming up | Defaults: cuts and closures, consultations, key decisions coming up |
| 5. Your first briefing | "Here's what's happening in your patch." Count, top 3 results with AI toplines and badges, up to 3 upcoming meetings mentioning your issues | `GET /api/onboarding/preview` using the shared search builder; < 2 s; if 30 days is empty, widen to 12 months with a note. Skeleton while loading |
| 6. Keep me posted | Proposed saved searches (one per topic/keyword group, scoped to councils + signals) with names, each toggleable; frequency (daily default, weekly, instant); "Invite colleagues" (emails, admins only) | Creates saved searches + subscriptions; sends invitations; sets `onboarded_at`; redirects to `/` |

After onboarding:
- Home shows a 3-item "Get the most from Minutes Search" checklist: open a summary's source; try an exact
  phrase search; share a saved search. Dismissible; completes automatically.
- Welcome email with the first briefing contents. Day-3 nudge only if no alert or briefing was opened.
- Admin console activation funnel: invited → signed in → onboarded → first saved search → returned within 7 days.

## 6. Account management

`/settings`: profile; area, topics and signals (re-runs the step components); notification hour; security
(passkeys, two-factor, sessions with "sign out everywhere"); organisation (members, invitations, roles for
admins); data ("Download my data" as JSON; "Delete my account": removes user, profile, subscriptions,
bookmarks, private saved searches; shared searches transfer to an org admin).
