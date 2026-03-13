# ADR-010: BullMQ + Redis for Async Job Queue

## Status
ACCEPTED

## Date
2026-03-13

## Context

Tools run asynchronously — a user triggers work from Telegram or the dashboard,
the tool executes in the background, results land in Postgres, and the user gets
notified. We need a queue that handles dispatch, execution, retries, status
tracking, and scheduled/recurring jobs.

Three options evaluated:

1. **Postgres SKIP LOCKED** — a `jobs` table polled by workers. Zero new infra
   but requires building retry logic, scheduling, priority, concurrency control,
   and status UI from scratch. Starts simple, gets ugly as tool count grows.

2. **BullMQ + Redis** — mature Node.js queue library backed by Redis. Built-in
   retries, scheduling, priority, concurrency, job chaining. Bull Board provides
   a free admin UI. Battle-tested (Shopify, GitLab, others).

3. **Inngest** — event-driven platform with declarative step functions and an
   excellent built-in dashboard. Most elegant DX for complex multi-step workflows.
   Requires running an Inngest server (self-hosted) or using their cloud.

## Decision

**BullMQ + Redis.**

### Why BullMQ over Postgres SKIP LOCKED

Every feature we need (retries, scheduling, priority, concurrency, status UI)
is built in. With Postgres we'd write and maintain ~500+ lines of queue
infrastructure instead of building tools. The abstraction earns its place.

### Why BullMQ over Inngest

Inngest's advantages — declarative step functions and a polished dashboard —
don't justify the additional infrastructure at current scale. BullMQ covers
our needs with a dependency (Redis) that's already justified for session
storage, caching, and rate limiting. If workflows become complex enough to
need Inngest's step orchestration, the tool functions stay the same — only
the dispatch wrapper changes. Migration path is clean.

### Redis justification

Redis isn't added solely for the queue. It serves multiple roles:

| Use | Why |
|-----|-----|
| BullMQ job queue | Dispatch, retry, schedule |
| NextAuth session store | Fast session lookups |
| Rate limiting | Per-user, per-tool throttling |
| Cache | Expensive query results, API responses |

One Redis instance, one Docker container, multiple uses.

### Usage patterns

**Basic tool dispatch:**

```typescript
import { Queue, Worker } from 'bullmq';

const toolQueue = new Queue('tools', { connection: redis });

// Worker processes jobs
const worker = new Worker('tools', async (job) => {
  const { toolName, params, userId } = job.data;
  const tool = toolRegistry.get(toolName);
  const result = await tool.run(params);
  await prisma.jobResult.create({
    data: { jobId: job.id, toolName, result, userId }
  });
}, { connection: redis });

// Trigger from anywhere (Telegram, dashboard, CLI)
await toolQueue.add('scrape', {
  toolName: 'scrappy_scrapper',
  params: { source: 'bizbuysell' },
  userId: 'operator',
});
```

**Scheduled/recurring jobs:**

```typescript
await toolQueue.add('daily-scrape', {
  toolName: 'scrappy_scrapper',
  params: { source: 'bizbuysell' },
}, {
  repeat: { cron: '0 8 * * *' }  // every morning at 8am
});
```

**Job chaining (scrape → score):**

```typescript
const flow = new FlowProducer({ connection: redis });

await flow.add({
  name: 'score-deals',
  queueName: 'tools',
  data: { toolName: 'deal_scorer' },
  children: [{
    name: 'scrape-listings',
    queueName: 'tools',
    data: { toolName: 'scrappy_scrapper', params: { source: 'bizbuysell' } },
  }],
});
// scrape runs first, score runs after scrape completes
```

**Retries with backoff:**

```typescript
await toolQueue.add('scrape', data, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 5000 },
});
```

**Admin UI (Bull Board):**

```typescript
import { createBullBoard } from '@bull-board/api';
import { BullMQAdapter } from '@bull-board/api/bullMQAdapter';
import { NextAdapter } from '@bull-board/next';

// Mount at /admin/queues in Next.js
createBullBoard({
  queues: [new BullMQAdapter(toolQueue)],
  serverAdapter: new NextAdapter(),
});
```

Provides job status, failure logs, retry controls, and queue metrics
behind NextAuth — no custom status page needed.

### Infrastructure

Added to `docker-compose.yml` (production) and `docker-compose.dev.yml` (local):

```yaml
redis:
  image: redis:7-alpine
  volumes:
    - redisdata:/data
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5
```

Resource footprint: ~10-30 MiB. Redis AOF persistence disabled by default
(queue state is ephemeral). Enable if job loss on Redis restart becomes a
problem.

## Consequences

### Positive
- Retries, scheduling, priority, concurrency — all built-in, zero custom code
- Bull Board gives free admin UI for job visibility
- Job chaining via FlowProducer handles multi-step workflows
- Redis serves multiple purposes (sessions, cache, rate limiting, queue)
- TypeScript-native with strong typing
- Battle-tested at scale far beyond our needs

### Negative
- Redis is an additional container to run and monitor (mitigated: Beszel
  tracks it, healthcheck in docker-compose, ~10 MiB footprint)
- Queue state is ephemeral by default — Redis restart loses pending jobs
  (mitigated: jobs are retriggerable, enable AOF if this matters)
- BullMQ is tied to Redis — can't swap to another backing store without
  changing libraries

### Risks
- Redis memory grows unbounded if jobs aren't cleaned up (mitigated: BullMQ
  `removeOnComplete` and `removeOnFail` options with configurable retention)
- Bull Board exposed without auth (mitigated: mount behind NextAuth middleware,
  operator-only access)
