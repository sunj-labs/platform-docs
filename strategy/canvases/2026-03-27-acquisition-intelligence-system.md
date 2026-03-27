# Acquisition Intelligence System — Three-Layer Deal Sourcing — Product Canvas

## Stage 1: Thesis

### Value Proposition (one sentence)

For **the family acquiring businesses under a 1031 deadline**, the acquisition intelligence system finds motivated sellers in growing markets before they list — using public data, owner behavior signals, and demographic tailwinds to surface off-market opportunities that competitors never see.

### Audience Scope

Scope: **Operator** (system output), progressing to **Domain** (repeatable for any acquisition searcher)

### The Problem

Current deal sourcing is reactive: wait for businesses to appear on BizBuySell/BizQuest, compete with every other buyer who sees the same listings. The conversion rate is 4.6% — 90 deals out of 1952 make it to family review.

The family needs **proactive sourcing**: identify businesses that match the acquisition criteria, in markets with demographic tailwinds, owned by people showing signs of wanting to sell — before those businesses are ever listed. The 1031 timeline (180 days to close) means the family can't wait for the market to bring deals to them.

### The Bet

Three layers of intelligence, stacked, produce acquisition candidates that no aggregator surfaces:

```
Layer 1: MARKET SELECTION (where to look)
  Demographics, migration, business formation → high-growth markets

Layer 2: SELLER IDENTIFICATION (who's there)
  PPP/FOIA data, state registrations, Google Maps → operating businesses

Layer 3: MOTIVATION SCORING (who'll sell)
  Owner age, social media dormancy, review velocity, listing age → sell probability
```

Broker sites become **comp validation** — calibrate multiples, find motivated listed sellers (180+ days, price cuts), but not primary deal flow.

---

## Stage 2: Shape

### Layer 1: Market Selection — Where to Look

**Goal**: Rank geographies by acquisition attractiveness before looking at individual businesses.

**Data sources** (all free/public):
| Source | Signal | What it tells you |
|--------|--------|------------------|
| Census ACS | Population growth, median income, age distribution | Is this market growing? Can residents afford services? |
| IRS SOI | Net migration by county (tax return address changes) | Are people moving in or out? |
| Census Building Permits | New housing permits | Leading indicator: 2-3 year demand growth |
| BLS QCEW | Employment by industry + county | Is the deal's industry growing here? |
| Census Business Patterns | Business formation/closure rates | Healthy vs dying economy |
| U-Haul migration data | One-way rental pricing | Real-time proxy for net migration demand |

**Output**: `DemographicProfile` per county/metro with a **marketTailwind** score (0-100).

**Enrichment beyond Census (your question about real-time/municipal)**:
- **City-level permit data**: Many municipalities publish building permits monthly (not just Census annual). Higher granularity.
- **Municipal budget/bond ratings**: Growing cities issue bonds for infrastructure. Moody's ratings are public.
- **School enrollment trends**: Growing enrollment = young families moving in = service demand.
- **USPS change-of-address data**: Aggregated net migration by zip code. More granular than IRS SOI.

### Layer 2: Seller Identification — Who's There

**Goal**: In high-tailwind markets, identify operating businesses that match acquisition criteria.

**Data sources**:
| Source | Signal | What it tells you |
|--------|--------|------------------|
| **PPP/FOIA** | Loan amount, NAICS, jobs, address | Business exists, has payroll, industry known. Loan ≈ 2.5x monthly payroll. |
| **State business registrations** | Entity name, registered agent, filing status | Business is active. Lapsed agent = possible neglect signal. |
| **Google Maps / Places API** | Business name, address, hours, reviews, phone | Business is operating. Review count/velocity = health signal. |
| **Secretary of State filings** | Annual report dates, officer names, formation date | Owner identity, business age, filing compliance |

**Output**: `AcquisitionCandidate` — a business in a target market that matches our NAICS/size criteria. NOT a listed deal — a potential seller we've identified from public records.

**PPP enrichment with recent data (your question)**:
- PPP data is 2020-2021. Cross-reference with current Google Maps presence (still operating?) and state registration (still active?).
- Owner address change (USPS): if the PPP loan address differs from current Google Maps address, the owner may have moved — life transition signal.

### Layer 3: Motivation Scoring — Who'll Sell

**Goal**: Among identified businesses, rank by likelihood the owner would sell if approached.

**Signals** (each adds to a motivation score):
| Signal | Source | Weight | Interpretation |
|--------|--------|--------|---------------|
| Owner age 60+ | LinkedIn, state filings (officer DOB where available) | High | Retirement window |
| Google review velocity declining | Google Maps API (reviews over time) | Medium | Owner disengaging |
| Social media dormancy | Business Facebook/Instagram last post date | Medium | Not investing in growth |
| Listing on broker site 180+ days | BizBuySell/BQ scrape data | High | Already trying to sell, struggling |
| Price cuts on listing | BizBuySell price history | High | Motivated, flexible on price |
| Job posting for GM pulled | Indeed/LinkedIn job history | Medium | Owner reconsidered stepping back |
| Lapsed registered agent | State SOS filings | Low | Administrative neglect |
| Local news: "retiring" | SERP monitoring | High | Public declaration of intent |

**Output**: **motivationScore** (0-100) per candidate. High = approach now.

### How broker sites fit

Broker sites are NOT primary deal flow. They serve three purposes:

1. **Comp validation**: What are similar businesses in this market priced at? What's the Price/SDE multiple?
2. **Motivated listed sellers**: Listing 180+ days + price cuts = owner who can't find a buyer. Approach with a credible offer.
3. **Market calibration**: How many businesses are listed vs. our off-market candidates? If 5 laundromats are listed in Charlotte but we identified 20 from PPP, the off-market opportunity is 4x the listed market.

### The scoring stack

A candidate scores high when ALL THREE layers align:

```
High tailwind market (Layer 1: marketTailwind 80+)
  + Operating business matching criteria (Layer 2: right NAICS, right size)
    + Owner showing sell signals (Layer 3: motivationScore 70+)
      = CALL THIS PERSON
```

### Outreach

Personalized, credible, family-oriented:

> "Dear [Owner], my family is looking to acquire a [industry] business in [city]. We've been researching the [city] market and believe it's an excellent location for long-term growth. We're backed by SBA 7(a) financing with significant equity available. If you've ever considered a transition, we'd welcome a confidential conversation."

Key differentiator: **family buyer** (not PE, not roll-up, not tire-kicker). Mom and Dad's care is the motivation. That resonates with small business owners who care about legacy.

---

## Stage 3: Commit

### Build order

| Phase | What | Depends on | Time |
|-------|------|-----------|------|
| **A** | DemographicProfile ingest (Census + IRS) | Nothing | 2-3 days |
| **B** | PPP/FOIA candidate ingest + filtering | Nothing (parallel with A) | 2-3 days |
| **C** | Google Maps verification (is business still operating?) | B | 1-2 days |
| **D** | Broker scraper for comp pricing | Nothing (parallel) | 3-5 days |
| **E** | Motivation scoring (owner signals) | B, C | 3-5 days |
| **F** | Outreach generation (Claude Haiku personalized letters) | B, E | 1-2 days |
| **G** | Operator review page (/outreach/candidates) | All above | 2-3 days |

A and B can start immediately in parallel. C-F build on them.

### Multi-tenant architecture

- **DemographicProfile** → platform-wide (shared, like Deal)
- **AcquisitionCandidate** → platform-wide (shared pool of identified businesses)
- **OutreachCampaign**, **OutreachLetter**, **MotivationScore** → tenant-scoped
- Scoring criteria (which NAICS, which markets, which signals) → via tenant's Search/InvestmentParameters

### Relationship to existing pipeline

```
EXISTING: Listed deals → Score → Review → Call Prep → Pursue
NEW:      Market → Candidates → Motivation → Outreach → Response → Pursue
SHARED:   Pursue → LOI → Due Diligence → Close (Phase 3)
```

Both funnels converge at "Pursue." The family reviews both listed deals and off-market candidates in the same Call Prep.
