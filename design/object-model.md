# Object Model

The shared nouns of the sunj-labs ecosystem. Same word everywhere — code, issues, Telegram, dashboard, Langfuse, database schemas. No synonyms.

## Core Objects

| Object | Description | Appears In |
|--------|-------------|------------|
| **User** | A person who interacts with the system (operator, collaborator, viewer) | Auth, dashboard, all apps |
| **Agent** | A domain-scoped assistant with a system prompt, tool access, and an intent classifier | Telegram, dashboard, dispatch layer |
| **Tool** | A callable capability with typed params, declared secrets, and a run function | Tool registry, job queue, subprocess worker |
| **Job** | An async execution of a tool (queued, running, completed, failed) | BullMQ, dashboard, Telegram |
| **Trace** | An observed LLM interaction (model, tokens, cost, latency) | Langfuse |
| **Deal** | A potential acquisition target | Deal pipeline, POA |
| **Candidate** | An SBA 7a business listing that qualifies | POA, deal pipeline |
| **Service** | A running infrastructure component (app, postgres, redis, langfuse) | Ops dashboard, healthchecks |
| **Deploy** | A production deployment (sha, status, duration) | CI/CD, ops dashboard |
| **Backup** | A database snapshot (filename, trigger, status) | Backup scripts, ops dashboard |
| **DailyCost** | Aggregated API spend for a given day | Cost tracking, ops dashboard |
| **Rule** | A safety or business constraint governing tool behavior | Tools, validation |
| **Task** | A unit of work tracked on the board | GitHub Projects, dashboard |

## Relationships

```
User --messages--> Agent (via Telegram or dashboard)
Agent --classifies intent--> Job (via IntentClassifier)
Job --validated by--> JobValidator (trust boundary)
Job --executes--> Tool (in isolated subprocess)
Tool --produces--> Trace (in Langfuse)
Tool --governed-by--> Rule
Tool --incurs--> DailyCost
Tool --evaluates--> Deal
Deal --qualifies-as--> Candidate (when SBA 7a criteria met)
Deploy --triggers--> Backup (pre-deploy)
Service --monitored-by--> Healthcheck
Task --references--> Deal | Tool | Job
```

## Trust Zones

```
UNTRUSTED: Agent + IntentClassifier
  - sees tool descriptions, param schemas
  - cannot access secrets or execute tools

TRUST BOUNDARY: JobValidator
  - validates tool exists, params correct, user authorized

TRUSTED: Tool subprocess
  - receives only its declared secrets
  - executes in isolated child process
  - writes results + traces
```

## Where Objects Live

| Object | Persistence | Owner |
|--------|------------|-------|
| User, Job, Deploy, Backup, DailyCost, Service | Postgres (Prisma) | ops app |
| Agent | Code (agent definitions with system prompt + tool scope) | ops app |
| Tool, Rule | Code (registry.ts, per-tool modules) | ops app |
| Trace | Langfuse (read via API) | Langfuse |
| Deal, Candidate | Postgres (Prisma) | ops app (future) |
| Task | GitHub Projects | GitHub (skinned via dashboard, future) |

## Role Scoping

| Role | Sees | Can do |
|------|------|--------|
| **OPERATOR** | Everything | Trigger tools, manage backups, view costs, admin |
| **COLLABORATOR** | Assigned scope (e.g. deal pipeline) | Run scoped tools, view scoped data |
| **VIEWER** | Read-only dashboards | View only |

## Rules for Object Evolution

When building a new feature, ask: *Can I express this using objects that already exist?*

New objects are expensive. Reuse is cheap. If you need a new object, it gets added here first — not invented ad hoc in a single repo.
