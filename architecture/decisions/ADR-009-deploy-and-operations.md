# ADR-009: Deployment, Backups, and Local Development

## Status
ACCEPTED

## Date
2026-03-13

## Context

With the full TypeScript stack (ADR-008), we need a clear deployment model that
covers three modes: local development (fast iteration), push-to-main production
deploys, and data safety. The existing CI pipeline (GitHub Actions → GHCR → EC2
via Tailscale) is documented but gaps exist around local dev orchestration,
database backups, and rollback procedures.

Vercel was considered but rejected for production: Langfuse, Postgres, Redis,
and Beszel are self-hosted. A split deploy (Vercel for app, EC2 for services)
adds complexity without reducing it. Everything on one box, one docker-compose,
one SSH target.

## Decision

### 1. Local Development — Fast Loop

One command to start everything locally:

```bash
docker-compose -f docker-compose.dev.yml up -d   # Postgres + Redis
npm run dev                                        # Next.js with hot reload
```

`docker-compose.dev.yml` runs **only the services the app depends on** — not
the app itself. The app runs on bare metal via `next dev` for hot reload.

```yaml
# docker-compose.dev.yml
services:
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: sunj_dev
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
    volumes:
      - pgdata_dev:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata_dev:
```

**Local dev workflow:**

```
Edit code → save → Next.js hot reloads (~200ms)
Edit Prisma schema → npx prisma migrate dev → instant schema update
Write a tool → test it → see Langfuse traces (if local Langfuse, or skip)
Run tests → vitest --watch (runs on save)
```

**Seed data:**

```bash
npx prisma db seed     # Prisma seed script with realistic test data
```

Seed script lives in `prisma/seed.ts`. Deterministic, idempotent, runnable
at any time. New developers (or future-you after a break) go from clone to
working app in under 5 minutes:

```bash
git clone → npm install → docker-compose -f docker-compose.dev.yml up -d → npx prisma migrate dev → npx prisma db seed → npm run dev
```

**Langfuse locally:** Optional. For most iteration, skip it. When debugging
traces, add Langfuse to `docker-compose.dev.yml` temporarily or point the
local app at the production Langfuse instance via Tailscale.

### 2. Production Deploy — Push to Main

```
git push origin main
       │
       ▼
GitHub Actions CI
├── 1. Lint (ESLint)
├── 2. Type check (tsc --noEmit)
├── 3. Secret scan (gitleaks)
├── 4. Test (vitest run)
├── 5. Build Docker image
├── 6. Push to GHCR
└── 7. Deploy to EC2
            │
            ▼
      SSH via Tailscale
      docker-compose pull
      docker-compose up -d --remove-orphans
      health check (curl /api/health, 3 retries)
      notify Telegram: "Deployed {sha} ✓" or "Deploy FAILED"
```

**Production `docker-compose.yml`:**

```yaml
services:
  app:
    image: ghcr.io/sunj-labs/app:latest
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://...
      REDIS_URL: redis://redis:6379
      NEXTAUTH_URL: https://...
      NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  postgres:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  langfuse:
    image: langfuse/langfuse:latest
    ports:
      - "3001:3000"
    environment:
      DATABASE_URL: postgresql://...
      NEXTAUTH_URL: http://langfuse:3000
    depends_on:
      postgres:
        condition: service_healthy

  beszel-hub:
    image: henrygd/beszel:latest
    ports:
      - "8090:8090"
    volumes:
      - beszeldata:/beszel_data

volumes:
  pgdata:
  redisdata:
  beszeldata:
```

**Migrations in production:**

Prisma migrations run as part of the deploy step, before the new app container
starts:

```bash
# In deploy script, after pull, before up
docker-compose run --rm app npx prisma migrate deploy
docker-compose up -d --remove-orphans
```

`prisma migrate deploy` applies pending migrations. It does not generate new
ones (that's `prisma migrate dev`, local only). If a migration fails, the
deploy stops and Telegram gets notified.

### 3. Rollback

Every deploy pushes a tagged image to GHCR: `ghcr.io/sunj-labs/app:{git-sha}`.
The `:latest` tag is also updated.

**Rollback procedure:**

```bash
# On EC2 via Tailscale SSH
# 1. Pin to previous known-good image
docker-compose pull   # if needed, or edit image tag
docker tag ghcr.io/sunj-labs/app:{previous-sha} ghcr.io/sunj-labs/app:latest
docker-compose up -d

# Or, more explicitly:
# Edit docker-compose.yml image tag to specific sha
# docker-compose up -d
```

**Database rollback:** Prisma migrations are forward-only by design. If a
migration breaks data, restore from backup (see below). This is why backups
run before every deploy.

### 4. Backups

**Postgres — automated daily + pre-deploy:**

```bash
#!/bin/bash
# scripts/backup-postgres.sh
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="sunj-backup-${TIMESTAMP}.sql.gz"

# Dump and compress
docker exec postgres pg_dump -U ${POSTGRES_USER} ${POSTGRES_DB} | gzip > /tmp/${BACKUP_FILE}

# Upload to S3
aws s3 cp /tmp/${BACKUP_FILE} s3://sunj-backups/postgres/${BACKUP_FILE}

# Clean up local
rm /tmp/${BACKUP_FILE}

# Prune S3 backups older than 30 days
aws s3 ls s3://sunj-backups/postgres/ | \
  awk '{print $4}' | \
  while read file; do
    date_str=$(echo $file | grep -oP '\d{8}')
    if [[ $(date -d "$date_str" +%s) -lt $(date -d "-30 days" +%s) ]]; then
      aws s3 rm s3://sunj-backups/postgres/$file
    fi
  done
```

**Schedule:**

| Trigger | When | What |
|---------|------|------|
| Cron | Daily 3:00 AM | Full pg_dump to S3 |
| Pre-deploy | Every deploy, before migration | Snapshot before schema change |
| Manual | On demand | `./scripts/backup-postgres.sh` |

**Restore procedure:**

```bash
# Download backup
aws s3 cp s3://sunj-backups/postgres/sunj-backup-{timestamp}.sql.gz /tmp/

# Stop app (prevent writes)
docker-compose stop app

# Restore
gunzip -c /tmp/sunj-backup-{timestamp}.sql.gz | \
  docker exec -i postgres psql -U ${POSTGRES_USER} ${POSTGRES_DB}

# Restart
docker-compose up -d
```

**Tested restore:** Run a restore to a scratch Postgres container monthly.
If you haven't tested a restore, you don't have backups — you have hope.

**Redis:** Not backed up. Job queue state is ephemeral. If Redis dies, pending
jobs are lost and can be re-triggered. If this becomes a problem, enable Redis
AOF persistence.

### 5. Deploy timing

| Action | Time |
|--------|------|
| CI (lint + typecheck + test + build + push) | ~3-4 min |
| Pre-deploy backup | ~30s |
| Migration | ~5s (usually no-op) |
| Container restart | ~10s |
| Health check pass | ~30s |
| **Total push-to-live** | **~5 min** |

## Consequences

### Positive
- Clone-to-running in under 5 minutes (local dev)
- Push-to-live in under 5 minutes (production)
- Automated backups with tested restore procedure
- Pre-deploy snapshots protect against bad migrations
- Tagged images enable quick rollback
- Telegram notification on every deploy (success or failure)
- Single docker-compose.yml defines the entire production stack

### Negative
- S3 backup adds AWS dependency (mitigated: could use any S3-compatible store)
- Single-box deployment means downtime during container restart (~10s)
- Forward-only migrations mean database rollback requires backup restore

### Risks
- EC2 disk fills up (mitigated: Beszel alerts on disk usage, backups go to S3 not local)
- Backup cron silently fails (mitigated: backup script sends Telegram on failure, monthly restore test)
- Migration runs on deploy and breaks production (mitigated: pre-deploy backup, test migrations locally first)
