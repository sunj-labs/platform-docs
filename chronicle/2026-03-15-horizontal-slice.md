---
session_id: 2026-03-15-horizontal-slice
date: 2026-03-15
duration: ~3 hours
repos: [sunj-labs/poa, sunj-labs/platform-docs]
tags: [deal-sourcing, scraping, scoring, email-digest, horizontal-slice, scraperapi, cheerio]
skills: [web-scraping, anti-bot-bypass, data-pipeline, email-design, sdlc-evolution, test-driven-development]
---

# Session: Horizontal Slice — Data Flowing End-to-End

## What Shipped

- **ScraperAPI + cheerio pipeline**: Pivoted from Playwright (blocked by Akamai) to ScraperAPI (residential proxy, $29/mo) + cheerio (TypeScript HTML parser). Full pipeline: ScraperAPI → cheerio → normalize → Postgres.
- **57 NC deals scraped and persisted** from BizBuySell search results.
- **Scoring engine (thin slice)**: 3 dimensions — price fit ($500K-$5M range), business maturity (yearEstablished), franchise penalty (-20 points). All 57 deals scored.
- **Live dashboard**: Deals sorted by score with color-coded badges (green/yellow/red), score bars, financial data.
- **Digest email**: Rich HTML email with pipeline summary, score distribution bar chart, deal cards with data completeness indicators, source links, broker info.
- **Exponential backoff + circuit breaker**: ScraperAPI client with jitter, 429 detection, Retry-After respect, circuit breaker after 5 failures.
- **43 unit tests**: Normalizer (parseCurrency, parseInteger, parseLocation), scraper (parseListingsFromHtml, parseDetailPage), scoring (all dimensions + overall), backoff calculation.
- **SDLC Rule 7**: Material backlog items trigger design artifact review — the ticket is the trigger, not the formality of the intake path.
- **5 new backlog items**: Email alert ingestion (#7), franchise/EBITDA scoring (#8), maturity filter (#9), auto-outreach engine (#10), detail enrichment gap (#11).

## What Was Learned

- **Bot detection is the real barrier to deal sourcing**, not parsing. BizBuySell uses Akamai CDN with aggressive fingerprinting. Headless Playwright gets blocked even with stealth options. Residential proxy networks (ScraperAPI, Bright Data) are the practical solution.
- **You can't build a residential proxy network on AWS.** EC2 IPs are datacenter IPs — Akamai flags them immediately. Residential means real ISP-assigned home IPs, which requires either an ISP partnership or a consumer app.
- **Sparse data changes scoring strategy.** When 0/57 deals have yearEstablished or EBITDA, maturity scoring is meaningless. Detail page enrichment is critical for scoring to be useful — but ScraperAPI free tier can't fetch detail pages. The $29/mo tier should fix this.
- **Horizontal slices beat vertical depth.** A thin slice through Scrape → Score → Dashboard → Digest proves the whole pipeline works. Going deep on any one agent first would have left us with a great scraper and nothing to show for it.
- **ToS risk is a strategic call.** BizBuySell prohibits scraping. ScraperAPI pushes compliance to the user. For personal deal sourcing (not data resale), the risk is accepted. Email alert ingestion (#7) is the compliant alternative.

## SDLC Evolution

- **Rule 7 added**: When a ticket introduces new objects, agents, state machines, or external integrations, it must include a "Design artifacts needed" checklist. The ticket itself is the trigger — not the formality of the intake path.
- **Working agreements added**: (1) Update GitHub tickets when features evolve during implementation. (2) Detect session boundaries automatically — run Reflect → Plan proactively.

## Collaboration Model

Human caught several strategic gaps: full detail scraping (not just top deals) produces better scoring across the whole set; ToS compliance is a deliberate risk decision not an oversight; the flywheel (data → analysis → communication → decision) matters more than optimizing any single step; material backlog items need design artifact triggers.

AI handled: anti-bot research, ScraperAPI/Apify/Bright Data comparison, cheerio pivot, scoring engine, digest template, test suite.

---

## Post Ideas

### 1. "The Bot Detection Tax: Why Your Business Acquisition Pipeline Needs a Residential Proxy Budget"

I tried to build a deal sourcing engine that scrapes business listing sites every 6 hours. Playwright with stealth mode, custom user agents, webdriver flag removal — none of it worked. Akamai's bot detection fingerprints headless browsers in milliseconds. The lesson: bot detection is a tax on automation, and the price is $29/month for a residential proxy network. You can't build your way around it with cleverness — you need traffic that looks residential because it is residential.

### 2. "Horizontal Slices Over Vertical Depth: Shipping a Complete Pipeline in One Session"

We had 7 agents to build. Instead of perfecting the scraper, we built a thin version of everything: scrape (57 deals), score (3 dimensions), display (dashboard with ranked scores), communicate (email digest with bar charts). In 3 hours, the family can see scored deals. In 3 weeks of building the perfect scraper, they'd see nothing. The lesson: prove the pipeline end-to-end, then deepen each step.

### 3. "When Your SDLC Meets Reality: Adding Rules Mid-Sprint Without Adding Ceremony"

Our SDLC has 6 rules. Today we added a 7th: material backlog items trigger design review. It came from a real moment — we created a ticket for an auto-outreach engine that introduces new objects, a new agent, and a new state machine. Without rule 7, someone would have started coding without updating the object model or state diagrams. The rule isn't "write a spec first" — it's "the ticket itself must identify which design artifacts need updating." The trigger is the ticket, not the ceremony.
