---
session_id: 2026-03-21-production-hardening
date: 2026-03-21
duration: ~2 hours (late night continuation)
repos: [sunj-labs/poa, sunj-labs/platform-docs]
tags: [production-incident, bug-fix, ux-polish, enrichment, process]
---

# Session: Production Hardening + Migration Incident

## What Shipped

### Production Migration Incident
- Discovered `prisma migrate deploy || true` had been silently swallowing migration failures since 2026-03-18
- Migration `0002_re_weight_absentee_deal_track` failed and blocked ALL subsequent migrations
- New SDE/enrichment fields never applied → dashboard raw SQL referencing `sde` column → 500 errors on API
- Fixed via SSH: `prisma migrate resolve --applied` then `prisma migrate deploy`
- #97 updated with incident details

### Deals Page Score Sort (#105)
- Raw SQL with `ORDER BY s.score DESC` replaces `orderBy: { createdAt: "desc" }`
- 84, 80, 77 score deals now appear on page 1 instead of buried on page 6+
- Same pattern as dashboard fix

### Financial Filter OR Logic (#106)
- CF + EBITDA filters now use OR (was AND — eliminated almost everything)
- SDE field included alongside CF in filter (BBS CF = SDE)
- Label: "Annual Income" → "Cash Flow (SDE)"

### 7 New Tickets
- #104 Deal lifecycle management epic
- #105-#107 Bug fixes (sort, filters, enrichment verification)
- #108 UX polish epic
- #109 Digest subscriptions
- #110 Chrome extension / manual enrichment research

### Design Notes
- #91 (ceremony) updated with deal view progression: Recommended → Working List → For Review → Below the Line
- #89 (search) upgraded to epic with Postgres FTS plan + full UX translation chain

## What Was Learned

- **`|| true` on migrations is a production outage waiting to happen.** It created the illusion of successful deploys for 2 days while the database schema diverged from the application code. The smoke tests passed because they only check `/api/health` and basic deal count — they don't query new columns.
- **Client-side sort with server-side pagination is a fundamentally broken pattern.** You fetch 50 deals ordered by date, sort them by score within the page, and think you're showing the best deals. You're not — you're showing the best of the 50 newest. The fix is always DB-level sort.
- **AND filters on alternative financial metrics eliminate everything.** Most deals have CF but not EBITDA (or vice versa). AND means "has both" which is almost nothing. OR means "has either" which is what the user actually wants.

---

## Post Ideas

### 1. "The Silent Migration Failure That Broke Production for Two Days"

We had 226 passing tests, a green CI pipeline, and successful smoke tests. Production was broken. The root cause: `prisma migrate deploy || true` — a single `|| true` that turned a database migration failure into a silent no-op. For two days, every deploy succeeded while the database schema quietly diverged from the application code. New columns referenced in queries didn't exist. The app served cached pages for some routes and 500 errors for others.

The fix took 30 seconds: SSH in, mark the failed migration as resolved, run deploy. The lesson took longer. Three concepts for engineering leaders: **silent failures are worse than loud failures** (a red build stops the line; a green build with swallowed errors creates false confidence), **smoke tests must test recent changes, not just baseline health** (our smoke test checked `/api/health` — it should have queried a recently-added column), and **the `|| true` pattern is a code smell in CI/CD** (it exists because "we'll fix it later" — but later never comes because the failure is invisible). For CTOs running due diligence: grep your deploy scripts for `|| true`. Each one is a silent failure waiting to become an incident.

### 2. "Why Your Deal Sourcing AI Sorts by Date Instead of Quality"

Our deal platform had 573 shortlisted deals, with the highest-scoring ones at 84 and 80. Users landed on the deals page and saw scores of 67, 57, 55. The best deals were on page 6. Why? The query fetched 50 deals `ORDER BY createdAt DESC`, then sorted them by score client-side. Classic pagination trap: you're sorting within a window, not across the dataset.

This pattern is everywhere in data products — especially AI-scored pipelines where the scoring happens asynchronously. The data arrives by date, the scores arrive later, and the UI shows "newest first" because that's what the default ORM query does. The user sees a pipeline that appears to be working (deals are flowing!) but the best results are buried. Three concepts: **sort by the signal you're optimizing for** (if score is the primary decision input, sort by score — not by when the data arrived), **client-side sort with server-side pagination is broken by design** (you can only sort what you've fetched), and **the default query is the product** (most users never touch filters — whatever page 1 shows IS their experience of your product). For product leaders: load your own product, don't touch any filters, and ask yourself: is page 1 showing the most valuable information? If not, your default query is wrong.
