# Broker Site Scraper — Independent Listing Discovery — Product Canvas

## Stage 1: Thesis

### Value Proposition (one sentence)

For **the operator who needs more evaluable deals before the 1031 clock expires**, the broker scraper discovers business listings on independent broker websites that never appear on BizBuySell, BizQuest, or BusinessesForSale — expanding the deal funnel with higher-quality, lower-competition listings.

### Audience Scope

Scope: **Operator**

Would I use this *this week*? Yes — 4.6% conversion rate means we need a wider top of funnel. Many quality deals are listed only on brokers' own websites.

### The Problem

Current sources (BBS, BQ, BFFS) are aggregators. Independent business brokers often list deals only on their own WordPress/HTML sites. These listings are higher quality (broker has a relationship with the seller) and lower competition (not on the main aggregators). Google can find them — "business for sale Charlotte NC site:*.com -bizbuysell" surfaces broker domains we're not scraping.

### The Bet

Google Search API discovers broker domains programmatically. For each domain, a Cheerio-first scraper extracts listings. Claude (Haiku) normalizes unstructured HTML into our deal schema. The result: 200-500 new deals per state that our competitors don't see.

### Conviction check

Operator scope: passes. The 1031 deadline makes every new deal source valuable. Broker sites have motivated sellers (they hired a broker) with real financials (broker prepared the listing).

---

## Stage 2: Shape

### One-Paragraph PR/FAQ

Five Pandas now discovers and ingests business listings from independent broker websites using Google Search API. The system finds broker domains per state, scrapes their listings, and uses Claude to extract structured deal data (price, revenue, SDE, industry, location). These deals enter the same scoring pipeline as BizBuySell and BizQuest listings — the family sees them alongside aggregator deals, ranked by the same criteria.

### Architecture (from Epic #147)

```
[Google Search API (Serper)] → discover broker domains per state
        ↓
[Broker Domain Registry] → dedupe against known sources
        ↓
[Cheerio Scraper (Playwright fallback)] → extract listing HTML
        ↓
[Claude Haiku Normalization] → structured data extraction
        ↓
[Dedup against existing deals] → hash on price + state + title similarity
        ↓
[Existing Scorer + Pipeline] → same scoring, same review page
```

### Users / Personas

1. **System** — discovers domains, scrapes listings, normalizes data
2. **Sanjay (Operator)** — monitors broker domain discovery, reviews new deals
3. **Family** — sees broker-sourced deals in Review alongside aggregator deals

### Access Scope

- [x] Operator manages broker domain list
- [x] All users see resulting deals (labeled "Broker: [firm name]")

### Key Outcomes

1. **Wider funnel** — 200-500 additional deals per discovery run
2. **Lower competition** — not on aggregators
3. **Higher quality** — broker-listed = motivated seller + real financials
4. **Source diversity** — reduces dependence on BBS/BQ/BFFS

### Cost model

| Component | Cost | Volume |
|-----------|------|--------|
| Serper API (search) | ~$0.004/query | 10 states × 5 queries = 50 queries/run = $0.20/run |
| ScraperAPI (page scrape) | existing budget | ~200 pages/run |
| Claude Haiku (normalization) | ~$0.001/listing | ~200 listings/run = $0.20/run |
| **Total per run** | **~$0.40** | Weekly runs = **$1.60/month** |

Well within budget.

### Query patterns

Per state, per industry cluster:
```
"business broker listings {state} site:*.com -bizbuysell -bizquest -businessesforsale"
"HVAC business for sale {state} broker"
"laundromat for sale {state} -bizbuysell"
```

Discover domains → scrape listing pages → normalize → ingest.

### What's deferred

- Playwright fallback for JS-rendered broker sites (Phase 2 — most are static WordPress)
- Automated weekly re-scrape (Phase 2 — monthly discovery sufficient initially)
- Broker relationship tracking (Phase 2 — track which brokers produce best leads)
- Login-wall bypass (Phase 2 — some brokers gate listings behind registration)

---

## Stage 3: Commit

### Risks & Assumptions

| Risk | Mitigation |
|------|-----------|
| Broker sites vary wildly in structure | Claude Haiku normalizes unstructured HTML — handles variation by design |
| robots.txt restrictions | Respect robots.txt. Skip domains that block crawlers. |
| Duplicate listings (broker lists on own site AND aggregator) | Dedup via title similarity + price + state hash — same as existing dedup |
| Low listing volume per broker site | Aggregate across 50-100 broker domains per state. Volume comes from breadth. |
| Scraper gets blocked | User agent rotation, randomized delays (2-5s between requests), respectful crawling |

### Constraints

- **Budget**: $1.60/month for weekly runs. Well within limits.
- **ScraperAPI budget**: Shares the $35/month pool. Broker scraping must not exceed 30% of budget.
- **Schedule**: Weekly discovery (new domains), monthly re-scrape (known domains). Not daily.

### Build plan (from Epic #147)

1. **Serper API integration** — discover broker domains per state, persist to `BrokerDomain` table
2. **Domain scraper** — Cheerio-first HTML extraction of listing pages from discovered domains
3. **Claude normalization** — Haiku extracts structured data: title, price, SDE, EBITDA, industry, state, description
4. **Dedup + ingest** — hash-based duplicate detection, insert as `source: BROKER_LISTING` deals
5. **BullMQ scheduling** — weekly discovery, monthly re-scrape, integrated into existing 3-queue architecture
6. **Monitoring** — yield per domain, LLM extraction accuracy, new domains discovered per run

### Relationship to PPP/FOIA

These are complementary funnels:
- **Broker scraper**: finds listed deals that aggregators miss (still listed, just on broker's own site)
- **PPP/FOIA**: finds off-market businesses that aren't listed anywhere (cold outreach)

Both feed into the same scoring pipeline. The family sees all deals in one view regardless of source.
