# ADR-008: Full TypeScript Stack

## Status
ACCEPTED

## Date
2026-03-13

## Context

The original architecture assumed Python for agents/tools and TypeScript for
frontends (ADR-004). This created a two-language system requiring:

- Separate API layer (FastAPI) to bridge Python tools and TypeScript dashboard
- Two ORMs or a shared-schema problem (SQLAlchemy vs. Prisma)
- Two runtimes, dependency managers, test frameworks, and deploy pipelines
- Cross-language contract maintenance

However: no Python code exists today. All tools are yet to be written. The
Anthropic TypeScript SDK is first-class. The Node ecosystem covers every
integration needed (scraping, Gmail, Telegram, external APIs). The operational
overhead of maintaining two languages for a solo/small-team operation outweighs
any ecosystem advantage Python provides when the primary workload is API calls
and data transformation — not ML training or statistical computing.

## Decision

Single-language TypeScript stack. Everything from database to UI in one
language, one runtime, one dependency manager.

### Core stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Framework | Next.js 14+ (App Router) | SSR, API routes, server actions in one framework |
| Language | TypeScript (strict) | End-to-end type safety, DB to UI |
| ORM + Migrations | Prisma + Postgres | Type-safe queries, declarative schema, built-in migrations |
| Auth | NextAuth.js | OAuth, magic link, credentials. Supports operator + family + collaborator access |
| LLM | Anthropic TypeScript SDK (`@anthropic-ai/sdk`) | First-class SDK. Claude calls inside tools, not a routing layer |
| Async jobs | BullMQ (Redis) or Inngest | Tool execution, retries, scheduling. Evaluate both before committing |
| Telegram bot | grammy or telegraf | TypeScript-native, same codebase as tools |
| Tracing | Langfuse TypeScript SDK | Same observability, same self-hosted Langfuse instance |
| Styling | Tailwind CSS + shadcn/ui | Per ADR-004, unchanged |
| Testing | Vitest + Playwright | Unit/integration (Vitest), E2E (Playwright) |
| Linting | ESLint + Prettier | Standard TypeScript tooling |

### Architecture

```
Interfaces
├── Web dashboard (Next.js SSR + client components)
├── Telegram bot (grammy/telegraf, same codebase)
└── CLI (optional, same TypeScript tools)
        │
        ▼
  Next.js API routes / Server Actions
        │
        ├── Direct tool calls (fast, synchronous)
        └── Job queue (async tools, long-running)
                │
                ▼
        ┌───────────────────┐
        │    Your Tools     │ ← TypeScript, all in one repo
        ├── ScrappyScrapper │
        ├── InboxClassifier │
        ├── DealScorer      │
        └── [next tool]     │
             │              │
             ├── Anthropic SDK (when tool needs LLM)
             ├── Prisma (read/write Postgres)
             └── Langfuse SDK (trace every LLM call)
        └───────────────────┘
```

### What this simplifies

| Before (two-language) | After (full TypeScript) |
|----------------------|------------------------|
| FastAPI + Next.js | Next.js only |
| SQLAlchemy + Alembic | Prisma |
| pytest + Vitest | Vitest only |
| ruff + ESLint | ESLint only |
| pip + npm | npm only |
| Python↔TypeScript API contract | Type-safe end-to-end |
| Two CLAUDE.md contexts per repo | One |

### Python escape hatch

If a future tool requires Python-specific libraries (ML, heavy statistics,
scientific computing), build it as a standalone service with an HTTP endpoint
or queue interface. Call it from TypeScript like any other external service.
This is a bridge built when needed, not infrastructure maintained from day one.

## Impact on existing ADRs

| ADR | Impact |
|-----|--------|
| ADR-002 (Langfuse) | No change to decision. SDK switches from Python to TypeScript. Self-hosted Langfuse unchanged. |
| ADR-004 (Frontend stack) | Reinforced. Next.js + TypeScript + Tailwind + shadcn/ui is now the entire stack, not just the frontend. |
| ADR-005 (Deploy tiers) | Revisit. Vercel becomes viable for the full app (not just static exports). Docker on EC2 remains an option for self-hosted requirements. |
| ADR-006 (Monitoring) | No change. Beszel + healthchecks + cost circuit breaker apply regardless of language. |
| ADR-007 (Dispatch) | Simplified. Intent classifier + tool registry + execution all in TypeScript. No cross-language dispatch. |

## Consequences

### Positive
- One language, one runtime, one mental model
- Type safety from database schema to UI component props
- Prisma migrations replace manual schema management
- Dramatically simpler CI/CD (one build, one deploy)
- Highest AI code generation quality (ADR-004 rationale applies to the full stack)
- Faster iteration — no context switching between languages
- Telegram bot shares code with dashboard tools (same functions, different interface)

### Negative
- Node.js is single-threaded — CPU-heavy tools need worker threads or external services
- Some niche Python libraries have no TypeScript equivalent (mitigated: escape hatch)
- Prisma has a learning curve if coming from raw SQL (mitigated: Prisma supports raw queries)
- BullMQ requires Redis (additional infrastructure) or use Inngest (managed) or Postgres-based queue

### Risks
- Anthropic TypeScript SDK falls behind Python SDK in features (mitigated: SDK is actively maintained, feature parity is a stated goal)
- Vitest + Playwright is less battle-tested than pytest for complex integration tests (mitigated: growing rapidly, adequate for this scale)
- Vendor lock-in if deploying to Vercel (mitigated: Next.js runs on Docker, can self-host)
