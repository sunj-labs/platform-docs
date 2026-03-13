# ADR-007: Agent Dispatch Architecture — Intent Classification + Deterministic Execution

## Status
EVALUATING

## Date
2026-03-13

## Context

The current architecture uses OpenClaw as an LLM-powered orchestrator: a user
message goes to an agent, the agent uses Claude to decide which tools to call,
and execution follows the LLM's plan. This raises concerns:

1. **Security surface.** An LLM deciding which tools to invoke means the routing
   layer itself is non-deterministic and harder to audit.
2. **Overhead.** Every dispatch requires an LLM call, even when the user knows
   exactly what they want ("POA, scrape BizBuySell").
3. **Reliability.** Tool execution is deterministic — the tools do what they're
   built to do. But LLM-routed orchestration can misroute, hallucinate tool
   names, or chain calls unnecessarily.
4. **Cost.** Routing calls consume tokens before any real work begins.

At the same time, full explicit dispatch (slash commands, exact tool names) puts
the burden on the operator to remember every tool and its parameters. The system
should be smarter than a CLI but more predictable than an autonomous agent.

## The Spectrum

```
Full LLM routing          Intent classification         Explicit dispatch
(OpenClaw today)          (proposed direction)          (slash commands)
│                              │                              │
│ LLM decides everything      │ LLM parses intent,           │ User specifies
│ Unpredictable, expensive    │ code executes deterministically│ everything
│ "Magic"                     │ Reliable, auditable           │ Rigid, forgettable
```

## Decision

Move toward **intent classification + deterministic execution**. This is not a
binary switch — it's a migration direction.

### How it works

```
User: "find new businesses on bizbuysell"
           │
           ▼
  Intent classifier (small/fast Claude call or local model)
  → { agent: "poa", tool: "scrappy_scrapper", params: { source: "bizbuysell" } }
           │
           ▼
  Validate: is this a known tool? Are params complete?
  → If ambiguous: ask the user to clarify
  → If clear: execute
           │
           ▼
  Tool execution (100% deterministic from here)
  → Tool calls Claude internally when needed (summarize, score, classify)
  → Writes results to Postgres
  → Writes traces to Langfuse
  → Returns structured result to user
```

### Key principle: LLM inside tools, not above them

The LLM's value is **within** tool execution (summarizing scraped listings,
scoring deals, classifying emails) — not in deciding which tool to call.
The dispatch layer is a thin classifier, not an autonomous planner.

### Interfaces are peers

All interfaces trigger the same dispatch:

```
Telegram message  ─┐
Dashboard action  ─┤→ Intent parser → Tool runner → Postgres/Langfuse → Response
CLI command       ─┘
```

No interface is privileged. Telegram, dashboard, and CLI are views into the same
system. The tool registry, execution engine, and data layer are shared.

### Tool registry contract

Each tool registers itself with:

```python
{
    "name": "scrappy_scrapper",
    "agent": "poa",
    "description": "Scrape business listings from a given source",
    "params": { "source": { "type": "str", "required": True } },
    "run": scrappy_scrapper_fn,
}
```

The descriptions power the intent classifier. The params power validation.
The registry is the single source of truth for what the system can do.

## What this replaces

- OpenClaw's LLM routing layer (when ready — no immediate removal)
- Ad-hoc agent-to-tool wiring
- Framework-specific tool registration patterns

## What this preserves

- OpenClaw can continue operating during transition
- All existing tools remain unchanged
- Langfuse tracing (traces originate from tools, not the dispatch layer)
- Monitoring stack (Beszel, healthchecks, cost circuit breaker)

## Relationship to other ADRs

- **ADR-002 (Langfuse):** Traces move from "agent made an LLM call" to "tool
  made an LLM call." Langfuse scope stays the same but trace ownership shifts
  to individual tools.
- **ADR-003 (Titans deferred):** Still deferred. The tool registry's descriptions
  and accumulated traces become training data if/when Titans is revisited.
- **ADR-006 (Monitoring):** Cost circuit breaker now monitors per-tool spend
  rather than per-agent spend. Same mechanism, finer granularity.

## Open questions

1. **Intent classifier implementation.** Small Claude call (haiku-class) vs.
   local model vs. regex patterns for common commands. Could start with regex
   + Claude fallback for fuzzy input.
2. **Tool chaining.** When one tool's output should trigger another, is that
   explicit in the tool definition or inferred? Leaning toward explicit
   (tool declares its "next steps" as suggestions, user confirms).
3. **OpenClaw transition.** Gradual migration vs. clean break. Current leaning:
   gradual — add intent classifier alongside OpenClaw, route simple commands
   through it, let OpenClaw handle complex/ambiguous requests until confidence
   builds.
4. **StrongDM.ai or similar.** Evaluate whether an existing platform solves
   this better than building it. Key criteria: self-hostable, Python-native,
   supports custom tool registration, doesn't impose its own LLM routing.

## Consequences

### Positive
- Predictable execution — same input produces same tool call every time
- Lower cost — one cheap classify call vs. multi-turn agent reasoning
- Auditable — dispatch log shows exactly what was parsed and executed
- Secure — tools are explicitly registered, no dynamic tool discovery
- Interface-agnostic — Telegram, dashboard, CLI are equal citizens

### Negative
- Less "magical" — complex multi-step requests need explicit chaining or user guidance
- Intent classifier is a new component to build and maintain
- Transition period with two dispatch mechanisms (OpenClaw + new classifier)

### Risks
- Intent classifier misparses a command and runs the wrong tool (mitigated:
  confirmation step for destructive actions, safety rules per tool)
- Tool registry becomes stale if tools aren't registered consistently
  (mitigated: CI check that all tools have registry entries)
