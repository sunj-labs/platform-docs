# ADR-011: Intent Classifier Abstraction + Credential Sandboxing

## Status
ACCEPTED

## Date
2026-03-13

## Context

Two related architectural concerns from ADR-007 (dispatch architecture):

1. **The intent classifier is the most likely component to be replaced.**
   Today it's a Claude API call. Tomorrow it could be a local model,
   StrongDM.ai, regex for common commands, or a fine-tuned classifier.
   Hardcoding the implementation couples the agent layer to one approach.

2. **The intent classifier and tool execution must be security-isolated.**
   The classifier parses natural language — an inherently untrusted input
   surface. If compromised via prompt injection, it must not be able to
   access credentials, execute tools directly, or exfiltrate secrets.
   Similarly, Tool A must not be able to read Tool B's credentials.

## Decision

### 1. Classifier Abstraction

All intent classification goes through a common interface:

```typescript
interface IntentClassifier {
  classify(
    message: string,
    agent: AgentDefinition,
  ): Promise<ClassifiedIntent>;
}

type ClassifiedIntent =
  | { type: "tool_call"; tool: string; params: Record<string, unknown>; confidence: number }
  | { type: "clarification"; question: string }
  | { type: "out_of_scope"; reason: string }
```

Implementations are swappable:

| Implementation | When |
|---------------|------|
| `ClaudeClassifier` | Default. Small Claude call with system prompt + tool descriptions. |
| `PatternClassifier` | Fast path for slash commands and structured input. No LLM. |
| `CompositeClassifier` | Try patterns first, fall back to Claude for fuzzy input. |
| `ExternalClassifier` | Wrapper for StrongDM.ai or similar. Future option. |

The classifier sees tool names, descriptions, and param schemas. It never
sees credentials, never executes tools, never writes to the database.

### 2. Trust Zones + Subprocess Isolation

The system is split into two zones with process-level enforcement:

```
┌──────────────────────────────────────┐
│  UNTRUSTED ZONE (main process)       │
│                                      │
│  Intent classifier                   │
│  Sees: message, tool descriptions,   │
│        param schemas                 │
│  Produces: { tool, params }          │
│  Cannot: access secrets, execute     │
│          tools, write to DB          │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  TRUST BOUNDARY (main process)       │
│                                      │
│  Job validator                       │
│  - Is this a registered tool?        │
│  - Are required params present       │
│    and correctly typed?              │
│  - Does user have permission?        │
│  - Within rate/cost limits?          │
│  → Enqueues validated job to BullMQ  │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  TRUSTED ZONE (child process)        │
│                                      │
│  Tool worker spawns subprocess       │
│  - Subprocess env contains ONLY      │
│    the secrets this tool declared    │
│  - Tool cannot read other tools'     │
│    secrets (they aren't in its env)  │
│  - Writes results to Postgres        │
│  - Writes traces to Langfuse         │
└──────────────────────────────────────┘
```

### Per-Tool Credential Scoping via Subprocess

Each tool declares what secrets it needs. The worker spawns a child
process with only those secrets in its environment:

```typescript
type ToolDefinition = {
  name: string;
  agent: string;
  description: string;
  params: Record<string, ToolParam>;
  requiredSecrets: string[];
  run: (
    params: Record<string, unknown>,
    secrets: Record<string, string>,
  ) => Promise<unknown>;
};
```

Execution spawns an isolated subprocess:

```typescript
async function executeTool(tool: ToolDefinition, params: Record<string, unknown>) {
  const secrets = await secretResolver.resolve(tool.requiredSecrets);

  // Child process gets ONLY declared secrets
  const result = await spawnToolProcess(tool.name, params, {
    ...secrets,
    DATABASE_URL: process.env.DATABASE_URL,
    LANGFUSE_PUBLIC_KEY: process.env.LANGFUSE_PUBLIC_KEY,
    LANGFUSE_SECRET_KEY: process.env.LANGFUSE_SECRET_KEY,
    LANGFUSE_BASE_URL: process.env.LANGFUSE_BASE_URL,
  });

  return result;
}
```

This means:
- Tool A's `GMAIL_OAUTH_TOKEN` is not in Tool B's environment
- Claude called inside a tool only sees that tool's scoped env
- A prompt injection in the classifier has zero access to any secrets
- A compromised tool has access to its own secrets only

### Secret Resolver Interface

```typescript
interface SecretResolver {
  resolve(secretNames: string[]): Promise<Record<string, string>>;
}
```

Upgrade path:

| Version | Implementation | Storage |
|---------|---------------|---------|
| v1 | `EnvSecretResolver` | `process.env` on the worker host |
| v2 | `DbSecretResolver` | Encrypted Postgres table, dashboard UI |
| v3 | `VaultSecretResolver` | HashiCorp Vault, AWS Secrets Manager, Infisical |

Each is a new implementation of the same interface. No tool code changes.

### What Each Layer Can Access

| Layer | Has access to | Cannot access |
|-------|--------------|---------------|
| Intent classifier | Tool names, descriptions, param schemas | Any secrets, DB write, tool execution |
| Job queue payload | Tool name, params, userId, metadata | Secrets (never serialized into job) |
| Job validator | Tool registry, user permissions, rate limits | Tool credentials |
| Tool subprocess | Its own declared secrets only | Other tools' secrets, main process env |
| Dashboard | Job results, costs, status (read) | Raw credentials |

## Consequences

### Positive
- Classifier is fully swappable without touching tools, workers, or UI
- Process-level isolation — tool cannot read another tool's secrets
- Prompt injection in classifier has zero blast radius
- Claude called inside a tool only sees that tool's scoped environment
- Secret resolver interface enables vault upgrade without tool changes
- Clear, auditable trust boundary

### Negative
- Subprocess spawning adds ~50-100ms latency per tool execution (negligible
  against typical tool runtimes of seconds)
- Per-tool secret declaration is manual (tool author must list required secrets)
- Subprocess communication requires JSON serialization in/out

### Risks
- Tool author declares more secrets than needed (mitigated: code review, registry audit)
- Subprocess crashes harder to debug than in-process (mitigated: structured logging, stderr capture)
- DATABASE_URL shared across tools that need DB (mitigated: acceptable now, per-tool DB users future option)
