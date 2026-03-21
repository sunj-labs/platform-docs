---
session_id: 2026-03-21b-first-digest-sent
date: 2026-03-21
duration: ~8 hours (marathon session)
repos: [sunj-labs/poa, sunj-labs/platform-docs]
tags: [enrichment, digest, scoring, ux, production-launch, family]
---

# Session: First Digest Sent to Family

## What Shipped

The flywheel spun for the first time. The family received deal sourcing emails.

### Enrichment Breakthrough
- Root cause: JS rendering + premium proxies = 100% timeout. Fix: renderJs=false, standard proxies, 45s timeout → **0% to 100% success rate**
- Burst mode (200/batch, 5min) cleared ~180 deals in hours
- Score invalidation: enricher now deletes stale scores and triggers re-scoring automatically
- Price floor raised from $200K to $500K (deals below don't produce enough income)

### Digest — Family-First Design
- 3 track groups: "1031 + Operating" (green), "Operating" (blue), "1031 Only" (amber)
- CF floor: only deals with $250K+ SDE/CF/EBITDA in top picks
- Per-group queries (1031 deals no longer crowded out by operating)
- "Top N" headers, visual distinction with colored borders
- Summary: "N new today", "M on shortlist", "K deals with family votes"
- Explanatory text on all surfaces (what's excluded and why)

### Scoring Page (née Rubric)
- Renamed "Rubric" → "Scoring" everywhere
- Financial glossary: Revenue → EBITDA → SDE → Cash Flow with bridge formula
- Dimension weights table: 1031 (RE 2x, CF 1.5x) vs Operating (CF 2x, RE 0.2x)
- Plain-language introduction: "Five Pandas searches business listing sites every day..."

### Production Launch
- 6 family users created (Rasik, Vaishali, Madhu, Ellen, Brahm, Lane)
- Welcome email template with logo, role explanation, sign-in CTA
- Send buttons on operator dashboard (no browser console needed)
- First welcome + digest emails sent to all 8 users
- Gmail send scope obtained via OAuth re-authorization

### UX Polish
- Sign-in page: trimmed logo (icon only, no duplicate text), sized to match heading
- PROD badge hidden in production
- Nav clean when unauthenticated
- Deals page: CF floor $250K, reset link, plain-language subtitle
- Dashboard: explanatory text for top matches

## What Was Learned

- **JS rendering costs 10x credits and causes 100% timeout on server-rendered pages.** BBS detail pages don't need it. This single setting change fixed the entire enrichment pipeline.
- **4 CI builds failed silently because one test broke.** A copy test expected "Why these deals?" but we changed it to "How we score →". No CI failure alert. The team didn't know production wasn't updating. Need CI failure notifications (#128).
- **The browser console is not a user interface.** When I told the operator to "run fetch() in the console," they asked "what console?" Built dashboard buttons instead. Always build the UI, never assume technical shortcuts.
- **Score-based sorting with server-side pagination is broken by design** unless you sort at the database level. Client-side sort within a page of 50 is sorting within a window, not across the dataset.
- **Temperance works.** When the operator said "temperance" after I was about to chase another UI fix, it reminded me that the family still hadn't received a single email. Stop polishing, start shipping.

---

## Post Ideas

### 1. "We Turned Off One Setting and Our AI Pipeline Went from 0% to 100%"

Our deal enrichment pipeline had been failing for a week. Every request timed out. We tried premium proxies, increased timeouts, added retry logic, built circuit breakers. Zero success rate. Then we turned off JavaScript rendering — a setting we'd enabled "just in case" — and every single request succeeded. The target pages were server-rendered HTML. JS rendering added 10-30 seconds of latency and cost 10x API credits to render JavaScript that didn't exist.

The lesson isn't about web scraping. It's about defaults that become invisible costs. Every "enable all the things" setting in your infrastructure is a bet: this capability might be needed, and the cost of having it on is low. But the cost compounds — in latency, in budget, in failure modes you can't diagnose because everything looks configured correctly. Three concepts: **audit your "just in case" settings periodically** (each one is a hypothesis that should be tested), **start with the minimum viable configuration** (add capabilities when you prove you need them, not preemptively), and **when debugging, remove features before adding them** (we spent hours adding retry logic when the fix was removing a checkbox). For engineering leaders: ask your team which settings are on "just in case." Each one is a silent tax.

### 2. "The First Email Is Worth More Than the Last Feature"

We spent 8 hours on a deal sourcing platform. We built enrichment pipelines, scoring algorithms, track-based grouping, financial glossaries, dimension weight tables, and pixel-perfect email formatting. Then at hour 7, the operator said "temperance" — and I realized the family still hadn't received a single email. Every hour we spent polishing was an hour the flywheel wasn't spinning.

We sent the email with broken logos, encoding artifacts in the subject line, and spacing issues in the summary. The family got 23 scored deals with SDE, Price/SDE ratios, and "View this deal" buttons. They could evaluate businesses. The formatting bugs became tickets for tomorrow. The deals were in their inbox tonight. Three concepts for product leaders: **the first email is the product** (not the dashboard, not the settings page — the push notification that reaches the user without them doing anything), **formatting bugs are tomorrow's work, missing functionality is today's** (a well-formatted email with no deals is worse than a rough email with 23 scored opportunities), and **"temperance" is the most valuable word in product development** (knowing when to stop building and start shipping is the difference between a product that launches and one that doesn't).
