# Application Stack Standard

## Decision

Full TypeScript stack. One language from database to UI.

See [ADR-008](../architecture/decisions/ADR-008-full-typescript-stack.md) for rationale.

## Stack

```
Next.js 14+ (App Router)
├── TypeScript (strict)   — end-to-end type safety
├── Prisma + Postgres     — ORM, migrations, type-safe queries
├── NextAuth.js           — auth for operator, family, collaborators
├── BullMQ + Redis        — async job queue, scheduling, retries
├── Anthropic TS SDK      — LLM calls inside tools
├── Langfuse TS SDK       — tracing, evals
├── Tailwind CSS          — design tokens map to utilities
├── shadcn/ui             — copy-paste components you own
└── Deploy: Docker on EC2 via Tailscale
```

## Project Structure

```
app/
├── (dashboard)/          — authenticated pages
│   ├── deals/
│   ├── tools/
│   └── admin/queues/     — Bull Board (operator-only)
├── api/
│   ├── health/           — healthcheck endpoint
│   ├── tools/            — tool trigger endpoints
│   └── auth/             — NextAuth routes
├── layout.tsx
└── page.tsx
src/
├── tools/                — tool implementations
│   ├── registry.ts       — tool registry (name, description, params, run fn)
│   ├── scrappy-scrapper.ts
│   ├── deal-scorer.ts
│   └── inbox-classifier.ts
├── lib/
│   ├── queue.ts          — BullMQ queue + worker setup
│   ├── langfuse.ts       — Langfuse client
│   └── anthropic.ts      — Anthropic client wrapper
└── components/
    └── ui/               — shadcn/ui components
prisma/
├── schema.prisma         — single source of truth for DB schema
├── migrations/           — Prisma migrations (committed to git)
└── seed.ts               — deterministic seed data
```

## Conventions

- Components live in `/src/components/ui/` (shadcn/ui, modified to use design tokens).
- Pages use App Router conventions (`/app/page.tsx`, `/app/layout.tsx`).
- Design tokens from `platform-docs/design/design-tokens.css` map to Tailwind config.
- No external component libraries beyond shadcn/ui. You own every component.
- Tools are registered in `src/tools/registry.ts`. Every tool is a typed function with metadata.
- Prisma schema is the single source of truth for database structure.
- Environment variables in `.env.local` (local) and Docker env (production). Never committed.

## Key Commands

```bash
# Local development
docker-compose -f docker-compose.dev.yml up -d   # Postgres + Redis
npm run dev                                        # Next.js hot reload

# Database
npx prisma migrate dev                             # Create/apply migration
npx prisma db seed                                 # Seed data
npx prisma studio                                  # Visual DB browser

# Testing
npx vitest                                         # Watch mode
npx vitest run                                     # CI mode

# Production deploy
git push origin main                               # Triggers full CI/CD pipeline
```
