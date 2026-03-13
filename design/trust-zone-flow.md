# Trust Zone Flow

Runtime sequence for tool execution through the three trust zones defined in ADR-011.

## Sequence

```
┌──────────┐    ┌───────────┐    ┌─────────────┐    ┌──────────┐    ┌───────────────┐    ┌────────────┐
│  User    │    │  Agent    │    │ JobValidator │    │ BullMQ   │    │ SecretResolver │    │ Subprocess │
│(Telegram/│    │(classifier│    │(trust       │    │(queue)   │    │(EnvResolver)  │    │(tool.run)  │
│Dashboard)│    │ only)     │    │ boundary)   │    │          │    │               │    │            │
└────┬─────┘    └─────┬─────┘    └──────┬──────┘    └────┬─────┘    └───────┬───────┘    └──────┬─────┘
     │                │                 │                │                  │                   │
     │  1. message    │                 │                │                  │                   │
     │───────────────>│                 │                │                  │                   │
     │                │                 │                │                  │                   │
     │  UNTRUSTED     │                 │                │                  │                   │
     │  ZONE          │                 │                │                  │                   │
     │                │ 2. classify()   │                │                  │                   │
     │                │  (sees: tool    │                │                  │                   │
     │                │   names, param  │                │                  │                   │
     │                │   schemas ONLY) │                │                  │                   │
     │                │                 │                │                  │                   │
     │                │ {tool_call,     │                │                  │                   │
     │                │  tool: "scrape",│                │                  │                   │
     │                │  params: {...}, │                │                  │                   │
     │                │  confidence: .9}│                │                  │                   │
     │                │─────────────────>                │                  │                   │
     │                │                 │                │                  │                   │
     │                │  TRUST          │                │                  │                   │
     │                │  BOUNDARY       │                │                  │                   │
     │                │                 │ 3. validate()  │                  │                   │
     │                │                 │ ├─tool exists? │                  │                   │
     │                │                 │ ├─params valid?│                  │                   │
     │                │                 │ ├─user authz?  │                  │                   │
     │                │                 │ └─rate limits? │                  │                   │
     │                │                 │                │                  │                   │
     │                │                 │ 4. enqueue()   │                  │                   │
     │                │                 │───────────────>│                  │                   │
     │                │                 │                │                  │                   │
     │                │                 │  TRUSTED       │                  │                   │
     │                │                 │  ZONE          │                  │                   │
     │                │                 │                │ 5. worker picks  │                   │
     │                │                 │                │    up job        │                   │
     │                │                 │                │                  │                   │
     │                │                 │                │ 6. resolve()     │                   │
     │                │                 │                │─────────────────>│                   │
     │                │                 │                │                  │                   │
     │                │                 │                │  {SCRAPER_KEY:   │                   │
     │                │                 │                │   "xxx"}         │                   │
     │                │                 │                │<─────────────────│                   │
     │                │                 │                │                  │                   │
     │                │                 │                │ 7. fork() with   │                   │
     │                │                 │                │    scoped env    │                   │
     │                │                 │                │──────────────────────────────────────>│
     │                │                 │                │                  │                   │
     │                │                 │                │                  │        8. tool.run│
     │                │                 │                │                  │         (params,  │
     │                │                 │                │                  │          secrets) │
     │                │                 │                │                  │                   │
     │                │                 │                │                  │        env has:   │
     │                │                 │                │                  │        SCRAPER_KEY│
     │                │                 │                │                  │        DATABASE_  │
     │                │                 │                │                  │        URL        │
     │                │                 │                │                  │        LANGFUSE_* │
     │                │                 │                │                  │                   │
     │                │                 │                │                  │        env lacks: │
     │                │                 │                │                  │        OTHER_TOOL │
     │                │                 │                │                  │        _SECRETS   │
     │                │                 │                │                  │        NEXTAUTH_* │
     │                │                 │                │                  │                   │
     │                │                 │                │   9. result      │                   │
     │                │                 │                │<──────────────────────────────────────│
     │                │                 │                │                  │                   │
     │                │                 │                │ 10. job.complete │                   │
     │                │                 │                │     (result)     │                   │
     │                │                 │                │                  │                   │
     │ 11. dashboard  │                │                │                  │                   │
     │     updates /  │                │                │                  │                   │
     │     telegram   │                │                │                  │                   │
     │     notified   │                │                │                  │                   │
     │<───────────────────────────────────────────────────                  │                   │
```

## Security Properties

1. **Classifier never sees secrets** — steps 1-2 operate with tool metadata only
2. **JobValidator is the single gate** — step 3 is the only path from untrusted to trusted
3. **Secrets resolved just-in-time** — step 6 fetches only this tool's declared secrets
4. **Subprocess isolation** — step 7 forks with a scoped env, no access to parent's full `process.env`
5. **No lateral movement** — tool A's subprocess cannot read tool B's secrets

## Implementation Map

| File | Role | Zone |
|------|------|------|
| `classifier.ts` | Intent classification | Untrusted |
| `agent.ts` | Agent registry + dispatch | Untrusted |
| `validator.ts` | JobValidator trust boundary | Boundary |
| `secrets.ts` | SecretResolver (scoped fetch) | Trusted |
| `tool-runner.ts` | Subprocess execution | Trusted |
| `worker.ts` | Child process entry point | Trusted (isolated) |
| `registry.ts` | Tool definitions + `requiredSecrets` | Shared (metadata) |

## References

- ADR-011: Classifier Abstraction & Credential Sandboxing
- SPEC-001: Ops Dashboard (trust zone flow section)
- Object Model: Trust Zones
