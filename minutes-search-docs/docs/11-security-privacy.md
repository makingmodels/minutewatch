# 11 · Security and privacy

## 1. Threat model (summary)

| Asset | Threat | Control |
| --- | --- | --- |
| User accounts | Account takeover, enumeration | Passwordless, single-use 10-minute links/codes, rate-limited auth routes, identical responses for any email, 2FA for owners/admins, passkeys |
| Private access | Unapproved sign-up | Closed sign-up enforced in a Better Auth user-creation hook (`08` §1) |
| Organisation data (saved searches, bookmarks) | Cross-organisation access (IDOR) | All access via DAL with actor checks; integration tests per DAL function for other-org access |
| Admin functions | Privilege escalation | Role checks in DAL and server actions; 2FA required; audit log |
| Upstream API | Overload, ban | Rate limits, circuit breaker, agreed volumes |
| AI pipeline | Prompt injection from documents; cost blow-up | Schema-only output, verification, plain-text rendering; daily budget cap |
| Web | XSS, CSRF, clickjacking | Sanitised snippets (`<mark>` only), React escaping, CSP, Better Auth CSRF, `frame-ancestors 'none'` |
| Secrets | Leakage | Vercel env vars, never in client bundles; `server-only` imports; secret scanning in CI |
| Email links | Token forgery/replay | HMAC-signed, expiring tokens; one action per token |

## 2. Access control rules

- Every server action and route handler: `const actor = await requireSession()` then calls DAL functions
  that take `actor`. No DAL function accepts a raw user ID from the client.
- Council data is readable by any signed-in user. App data rules:
  - Saved search: owner can read/write; org members can read and follow if `organization_id` = their org;
    org admins can edit/delete shared ones.
  - Bookmarks, profile, subscriptions: owner only.
  - Admin console: platform admin only.
- Database roles: `app_web` has SELECT on council tables and CRUD on app tables only; `app_worker` writes
  council and pipeline tables. Neither can drop tables.

## 3. Rate limits

| Route | Limit |
| --- | --- |
| Auth: request link/code | 5 per email per hour, 20 per IP per hour |
| Request access | 3 per IP per hour |
| Search | 60 per user per minute |
| Results digest | 20 per user per hour |
| Export | 10 per user per day |
| Onboarding preview | 30 per user per hour |

Implement in Postgres (sliding window table) or Vercel firewall rules for IP limits.

## 4. Headers

`Content-Security-Policy` (default-src 'self'; script-src 'self' with nonces; img-src 'self' data:;
connect-src 'self' + Sentry; frame-ancestors 'none'), `Strict-Transport-Security`, `X-Content-Type-Options:
nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` minimal.

## 5. Audit log

Record: sign-in, sign-out-everywhere, invitation sent/accepted/revoked, role changes, access request
decisions, exports, account deletion, admin job retries, impersonation start/stop. Retain 12 months.

## 6. Privacy (UK GDPR)

- Personal data held: name, email, job title, organisation, preferences, saved searches, bookmarks,
  usage logs. Lawful basis: legitimate interests / contract with the organisation.
- Council data is public-sector information; councillor names appear as published by councils.
- No user data is sent to the Claude API; only council text, queries and result content.
- Processors: Vercel, Neon, Anthropic, Resend, Sentry (list in the privacy notice).
- Users can download their data (JSON) and delete their account from settings.
- Logs: no email addresses in application logs; Sentry scrubs PII.
- Search query logging for product improvement: store query and filters with user ID hashed; 90-day retention.

## 7. Review before launch (T5.1)

Run a dependency audit, secret scan, OWASP ASVS level 1 checklist on auth/session/access control, the
IDOR integration tests, and a manual test of every public route.
