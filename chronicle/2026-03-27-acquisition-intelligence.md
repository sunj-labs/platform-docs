---
session_id: 2026-03-26-to-28-acquisition-intelligence
date: 2026-03-26 through 2026-03-28
duration: ~18 hours (multi-day session)
repos: [sunj-labs/poa, sunj-labs/platform-docs]
tags: [dedup, visual-harmonization, my-picks, search, freshness, market-tailwind, census, multi-tenancy, architecture, ux-translation, release-notes]
---

# Session: Acquisition Intelligence — From Deal Sourcing to Market Intelligence

## What Shipped

### 1. Deal Dedup Phase 2 (PR #197)
- Link-and-hide strategy: 367 duplicate groups merged, 376 aliases hidden
- Canonical deal selection by field completeness, data backfill from aliases
- `canonicalDealId IS NULL` filter across all 9 query surfaces
- Source badge: "BizBuySell, BizQuest (most complete)"
- Worker auto-merges new duplicates hourly

### 2. Visual Harmonization (PR #202, #209, #217 + direct pushes)
- 3 shared components: ScoreBadge, FinancialSnapshot, PlusesWatchOuts
- Applied to all 4 deal views (deals detail, review detail, call prep editor, brief)
- Pluses/Watch Outs: callout card with left accent bar (green/amber)
- All badges: outline style (gray border, colored text, no fill)
- 1031 Exchange: gray (classification). SBA Qualified: green (advantage)
- Simplified borders: all sections use neutral border-border

### 3. My Picks + Search + Freshness (PR #209)
- DealFavorite model: star deals with one tap, My Picks tab
- Family Picks tab (operator): see what family starred
- Propose for Call: one-tap proposal on starred deals
- DealSearch: type-ahead (300ms debounce) on Review + Deals pages
- FreshnessBadge: New (<7d), 2w ago, Mar 3 on deal rows
- Default tab by role: family → Not Yet Reviewed, operator → All

### 4. Market Tailwind — Census Intelligence (PR #227)
- 733 county profiles across 10 target states from Census ACS API
- Tailwind score (0-100): income, homeownership, poverty, education
- Processing schema (processing.*) separate from app DB
- Bridge worker populates Deal.marketTailwindScore
- 2,040 deals scored: 37 Strong Growth, 516 Growing, 1,474 Stable
- Wind icon + score on deal rows, detail pages, Call Prep cards
- Rubric page: two grouped sections (Deal Score + Market Tailwind) with formulas

### 5. Washington State Added
- WA added to DEFAULT_STATES via tenant-params facade

### 6. Release Notes Pipeline
- Daily release notes to family (23:30 UK time)
- Session-end hook prompts to draft notes after deploys
- release-notes-sender agent: reads markdown, converts to HTML, sends via Gmail
- First notes sent to 4 family members

### 7. Architecture: Multi-Tenancy Foundation
- getTenantParams() facade: all hardcoded thresholds flow through one function
- 6 multi-tenancy epics filed (#218-#223)
- Multi-tenancy hook: warns on hardcoded thresholds, missing tenant context
- tsc --noEmit pre-push hook: catches type errors before CI
- Interface Contract Protocol: caller contracts, model processors, contract tests

### 8. Acquisition Intelligence System Canvas
- Three-layer sourcing strategy: Market → Sellers → Motivation
- PPP/FOIA canvas: off-market deal discovery from public loan data
- Broker scraper canvas: independent broker site discovery via SERP
- Demographic market validation canvas: Census + IRS + BLS data
- Financial imputation engine: PPP loan → payroll → revenue → SDE → EBITDA
- IndustryBenchmark model: configurable per-NAICS ratios from multiple sources

### 9. Architect Review (2026-03-27)
- CONDITIONAL: 0 critical, 2 high, 4 medium
- H1 fixed: DealFavorite transfer in dedup merge
- H2 fixed: Star API e2e tests
- M1 fixed: CallPrepDeal delete+rerank in transaction
- M2 fixed: Review SQL LATERAL subquery for score versions
- M3 fixed: Review SQL LIMIT 200
- Suspense boundary fix: DealSearch without Suspense caused double-click nav

## What Was Learned

**Contracts are discovered by the second consumer.** Every shared component worked for its first use. Bugs appeared when the second page connected — Suspense missing, star relation not in merge logic, type mismatch between schema and client. The fix: write caller contracts before consumers exist, not after bugs teach them to you.

**Market intelligence changes deal evaluation.** The family was evaluating businesses in a vacuum — good deal vs bad deal. Adding market tailwind changed the question to "good deal in a good market?" The Charlotte HVAC at score 73 in a tailwind-87 market is fundamentally different from the Alabama laundromat at score 68 in a tailwind-22 market. Data the family can see.

**Processing data needs its own schema.** Census profiles (733 rows) and PPP candidates (will be 40K+) don't belong in the transactional app DB. Separate processing.* schema in the same Postgres instance gives logical isolation now, physical isolation later.

**Release notes are a contract with stakeholders.** The family receives features without context. Stars appear, badges change, search bars show up — nobody explains why. Daily release notes in family-friendly language close that loop. Written for Mom (76, phone). If she can understand it, everyone can.

## Post Ideas

1. **"Two Numbers That Changed How Our Family Evaluates Businesses"** — We had a deal scoring system that ranked businesses on 7 dimensions. It worked. Then we added a second number — market tailwind — that scores the location, not the business. Same business, different score depending on whether it's in Charlotte (growing) or rural Alabama (declining). The family's first reaction: "Why didn't we have this before?" Three concepts: dual-score evaluation, demographic intelligence, the difference between a good deal and a good deal in a good market.

2. **"The Contract Your Code Doesn't Know It Has"** — Every shared component has an implicit contract with its callers. The search bar needed a Suspense boundary. The merge function needed to transfer star data. The formatter needed a sanity cap. Each worked perfectly — until the second consumer connected. The fix isn't more testing. It's writing the contract before the consumer exists. Three concepts: interface contracts, boundary bugs, declaring requirements before they break.

3. **"PPP Loans as Acquisition Intelligence"** — 12 million businesses took PPP loans during COVID. Each loan record is public: business name, address, industry, loan amount (which correlates to payroll). A $500K PPP loan means ~$2.4M annual payroll. Cross-reference with Census demographics and you have acquisition candidates that no listing site surfaces. Three concepts: public data as private intelligence, payroll as revenue proxy, off-market vs on-market sourcing.

4. **"Daily Release Notes Written for Mom"** — Every feature we ship gets a release note written for a 76-year-old reviewing businesses on her phone. "Tap the ★ to save deals you like." Not "DealFavorite model with optimistic state updates." If she can understand it, the feature is well-designed. If she can't, we haven't finished designing it. Three concepts: stakeholder communication as design constraint, the Mom test, transparency builds trust.

5. **"The Processing Schema: Keeping Your App Database Fast"** — We needed to ingest 733 county demographic profiles and will soon ingest 40,000 PPP loan candidates. Putting that in the app's transactional database would slow every page load. Instead: a separate processing schema in the same Postgres instance. The app DB only receives scored outputs. Same server, different schema, clean boundary. Three concepts: schema isolation, processing vs transactional data, facade-based extraction.
