# Minutes Search

A private research tool for charities, campaign groups, unions and local journalists. It mirrors
every UK council meeting, decision and document from the Council Gateway API (about 1.66 million
documents from 320 councils), cleans and labels them, writes cited AI summaries, and lets invited
organisations search, read and set alerts.

> **Build agents:** start with [`knowledge.md`](knowledge.md), then [`PROGRESS.md`](PROGRESS.md).

## What it does

- **Complete, clean coverage**: every document in the API, synced hourly, with corrected dates, document
  types, agenda-item sections and duplicates removed.
- **AI summaries**: a cited summary of every document, a brief for every meeting and decision, and a
  digest of any search results.
- **Fast search**: keyword and phrase search over the full corpus, filtered by council, area, committee,
  date, document type, topic and signal (cuts, closures, consultations, contracts, grants).
- **Saved searches**: one click to save, share within an organisation and get email or in-app alerts.
- **Private access**: invite-only sign-in, organisations with roles, guided onboarding that ends with a
  personal briefing.

## Documentation map

| Doc | Covers |
| --- | --- |
| [knowledge.md](knowledge.md) | Rules, stack, commands and definition of done for whoever builds this |
| [PROGRESS.md](PROGRESS.md) | Build status, log, verified facts, open questions |
| [01 Product spec](docs/01-product-spec.md) | Users, scope, user stories with acceptance criteria, glossary |
| [02 Architecture](docs/02-architecture.md) | Components, data flow, package boundaries |
| [03 Council Gateway API](docs/03-council-gateway-api.md) | Upstream endpoints, measured behaviour, quirks |
| [04 Data pipeline](docs/04-data-pipeline.md) | Ingestion, cleaning, classification, segmentation, de-duplication |
| [05 Data model](docs/05-data-model.md) | Full Drizzle schema, indexes, sizing |
| [06 Search](docs/06-search.md) | Query parsing, ranking, snippets, facets |
| [07 AI summaries](docs/07-ai-summaries.md) | Prompts, output schemas, verification, batching, evaluation, cost |
| [08 Auth and onboarding](docs/08-auth-and-onboarding.md) | Invite-only access, roles, sign-in, onboarding flow |
| [09 Saved searches and alerts](docs/09-saved-searches-and-alerts.md) | Matching, delivery, email |
| [10 UX spec](docs/10-ux-spec.md) | Screens, states, copy, design tokens, accessibility |
| [11 Security and privacy](docs/11-security-privacy.md) | Threat model, access control, UK GDPR |
| [12 Testing and quality](docs/12-testing-and-quality.md) | Test strategy, golden sets, performance budgets |
| [13 Deployment and operations](docs/13-deployment-ops.md) | Environments, env vars, CI, cron, monitoring, runbooks |
| [14 Build plan](docs/14-build-plan.md) | Ordered tasks with acceptance criteria |
| [15 Decisions](docs/15-decisions.md) | Settled architecture decisions and why |
| [Reference data](docs/reference/README.md) | OpenAPI spec, real API responses, real document texts |

## Quick start (once built)

```bash
cp .env.example .env.local      # fill in values
pnpm install
pnpm db:up && pnpm db:migrate
pnpm db:seed                    # 3 councils through the real pipeline
pnpm dev                        # http://localhost:3000
pnpm worker                     # in a second terminal
```

## Stack

Next.js · TypeScript · Better Auth · Drizzle ORM · Postgres (Neon via Vercel; Docker locally) · pg-boss ·
Claude API (Haiku 5.5, Sonnet 5.5) · Resend · Tailwind + shadcn/ui · Vitest + Playwright.
