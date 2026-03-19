---
session_id: 2026-03-17-multitenancy-hooks
date: 2026-03-17
duration: ~4 hours
repos: [sunj-labs/poa]
tags: [multi-tenancy, sdlc-hooks, gmail-ingestion, personas, permissions]
skills: [prisma-schema, next-auth, bullmq, sdlc-process, hook-design]
---

# Session: Multi-Tenancy, Gmail Ingestion, and SDLC Hooks

## What Shipped

- **Multi-tenancy schema** — Tenant, TenantMember, Thesis, ThesisWeight models. Five Pandas and POA as separate tenants.
- **Gmail email ingestion pipeline** — classify incoming emails (BBS alert, BQ alert, broker deal sheet, noise), parse deal data, upsert to shared pool. BullMQ scheduled every 15 min.
- **Persona-aware copy system** — OPERATOR (technical), COLLABORATOR (plain English), STUDENT (kid-friendly), ADVISOR (professional), INTERN (task-oriented). Fallback chains per persona.
- **Fine-grained permissions** — string array on User model: view_deals, vote, comment, outreach, prune, manage_searches, manage_theses, manage_members, admin.
- **SDLC hooks** — pre-build gate (Edit/Write), pre-commit gate (Bash), bug diagnosis (UserPromptSubmit), tool failure diagnosis (PostToolUse), session start/end.
- **Deploy fixes** — Docker image prune, GHCR auth with dedicated PAT, force-recreate containers, sed compose SHA tags.

## What Was Learned

- **SDLC hooks need enforcement, not just reminders** — hooks that say "you should" get ignored. Hooks that say "STOP — write this section before proceeding" are more effective.
- **PostToolUse diagnosis hook** catches errors Claude discovers (not just user-reported). Essential for preventing brute-force debugging.
- **Multi-tenancy must be designed before data flows** — adding Tenant model after 600+ deals means migration + backfill.
- **Persona copy system** needs fallback chains: STUDENT → COLLABORATOR → OPERATOR. Don't duplicate strings for every persona.

---

## Post Ideas

### 1. "Building a Multi-Tenant Schema Before You Have Multiple Tenants"

We added Tenant, TenantMember, and Thesis models on day 4 — with only one tenant. Why? Because retrofitting multi-tenancy after data flows is a migration nightmare. The schema cost us 2 hours. Retrofitting would have cost 2 days. Design the isolation boundary early, enforce it later. The cost of the abstraction is cheap; the cost of the migration is not.

### 2. "Five Personas, One Codebase: How We Write Copy for a Family Business Tool"

Mom needs plain English. The operator needs technical precision. The nephew needs kid-friendly language. The CPA needs professional terminology. We built a persona-aware copy system with fallback chains — STUDENT falls back to COLLABORATOR falls back to OPERATOR. Same UI, different voice. 146 lines of code, zero duplication.

### 3. "SDLC Hooks That Actually Work: From 'You Should' to 'You Must'"

We tried advisory hooks first: "you should run tests before committing." They got ignored under time pressure. Then we rewired them as blocking gates: "STOP — write your verification section before this commit proceeds." The difference between a suggestion and a gate is the difference between knowing the process and following it.
