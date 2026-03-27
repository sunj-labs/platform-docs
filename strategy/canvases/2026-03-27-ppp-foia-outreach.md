# PPP/FOIA Data — Off-Market Deal Discovery & Cold Outreach — Product Canvas

## Stage 1: Thesis

### Value Proposition (one sentence)

For **the operator sourcing acquisition targets before the 1031 clock expires**, PPP/FOIA data reveals operating businesses with proven payroll in target states and industries that aren't listed on any marketplace — enabling cold outreach to motivated sellers before competitors know they exist.

### Audience Scope

Scope: **Operator**

Would I use this *this week*? Yes — the property sale is imminent. 1031 clock gives 45 days to identify, 180 days to close. Current pipeline is 4.6% conversion from listed deals. Off-market outreach opens a parallel channel.

### The Problem

Current deal flow comes from three listing aggregators (BizBuySell, BizQuest, BusinessesForSale). Every buyer in the market sees the same listings. Competition is highest on listed deals. Meanwhile, ~12 million businesses received PPP loans during COVID — public record via SBA FOIA release. Each loan record contains: business name, address, NAICS code, loan amount (correlates to 2.5x monthly payroll), jobs retained, lender. A business with a $500K PPP loan has ~$200K/month payroll = ~$2.4M annual revenue. That's a meaningful acquisition target.

Most of these businesses are NOT listed for sale. But the owner who took a PPP loan in 2020 is now 5 years into post-COVID operations. They may be tired, approaching retirement, or open to a conversation — especially if approached with a credible family buyer backed by SBA financing.

### The Bet

If we cross-reference PPP loan data with our target parameters (states, NAICS codes, loan size → implied revenue), we can identify off-market acquisition candidates and generate personalized outreach. This is a proprietary signal — nobody else in the deal sourcing space is doing this for small business M&A.

### Conviction check

Operator scope: passes. The 1031 clock makes parallel sourcing channels urgent. PPP data is free (public FOIA). The cost is compute + outreach, not data acquisition.

---

## Stage 2: Shape

### One-Paragraph PR/FAQ

Five Pandas now sources acquisition targets from public PPP loan records. The system identifies businesses in our target states and industries with loan amounts implying $250K+ annual payroll — businesses big enough to acquire. For each candidate, we generate a profile: estimated revenue, industry, location, years in operation. The operator can review candidates and draft personalized outreach letters. This is off-market deal flow — businesses that aren't listed anywhere but match our acquisition criteria.

### Users / Personas

1. **Sanjay (Operator)** — reviews PPP candidates, approves outreach, manages responses
2. **System** — ingests PPP data, filters by target parameters, generates candidate profiles, drafts outreach

### Access Scope

- [x] Operator only — cold outreach is an operator function, not family-facing

### Key Outcomes

1. **Parallel sourcing channel** — off-market deals alongside listed deals
2. **Proprietary signal** — no other buyer is using PPP data for acquisition targeting
3. **Higher quality leads** — PPP loan proves the business was operating and had real payroll
4. **Lower competition** — off-market = no bidding war

### Data source

**SBA PPP FOIA dataset**: https://data.sba.gov/dataset/ppp-foia

Key fields:
- `BorrowerName` — business name
- `BorrowerAddress`, `BorrowerCity`, `BorrowerState`, `BorrowerZip` — location
- `NAICSCode` — industry classification (matches our NAICS ontology)
- `CurrentApprovalAmount` — loan amount (2.5x monthly payroll)
- `JobsRetained` — employee count at time of loan
- `LoanStatus` — Active/Paid in Full/etc.
- `BusinessType` — LLC, Corp, Sole Prop, etc.
- `DateApproved` — when loan was granted

### Filtering strategy

```
PPP dataset (~12M records)
  → Filter: target states (NC, SC, VA, GA, FL, TN, AL, MD, DE, WA)
  → Filter: target NAICS codes (from our ontology — HVAC, plumbing, laundromat, car wash, storage, etc.)
  → Filter: loan amount $100K-$2M (implies $480K-$9.6M annual payroll — our acquisition range)
  → Filter: loan status = Paid in Full (business survived COVID)
  → Dedupe against existing Deal pipeline (same business name + state)
  → Result: ~5,000-20,000 candidates
```

### Enrichment

For each candidate, derive:
- **Estimated annual revenue**: loan amount × (12 / 2.5) = annual payroll. Revenue ≈ 3-5x payroll for service businesses.
- **Industry classification**: NAICS code → our ontology → AI resistance, necessity scores
- **Business age**: cross-reference with state business registrations (stretch)
- **Contact info**: Google Maps API → phone, website, owner name (stretch)

### Outreach

Personalized letter template:
> "Dear [Owner], my family is looking to acquire a [industry] business in [city/state]. We're backed by SBA 7(a) financing with [down payment] available for the right opportunity. We noticed your business has been operating successfully for [years]. Would you be open to a confidential conversation about whether a transition might be of interest?"

Key: this is NOT spam. It's a family buyer with real financing making a personal inquiry. Tone matters.

### What's deferred

- Automated outreach sending (Phase 2 — operator reviews and sends manually first)
- Google Maps enrichment (stretch — adds cost)
- Business registration cross-reference (stretch — per-state APIs vary)
- Integration into Deal pipeline (PPP candidates become Deals once owner responds)

---

## Stage 3: Commit

Operator scope — spec directly.

### Risks & Assumptions

| Risk | Mitigation |
|------|-----------|
| PPP data is 5 years old — businesses may have closed | Filter for "Paid in Full" status (survived). Cross-check with Google Maps existence (stretch). |
| Low response rate on cold outreach | Expected 2-5% response rate. 10,000 candidates → 200-500 responses → 20-50 conversations → 2-5 viable deals. That's a good funnel. |
| Privacy/legal concerns with cold outreach | PPP data is public record (FOIA). Outreach is a legitimate business inquiry, not marketing. Follow CAN-SPAM for email. |
| Data volume — 12M records | Bulk download, filter locally. One-time ingest, periodic refresh unnecessary (PPP program ended). |

### Constraints

- **Budget**: PPP data is free. Google Maps API for enrichment has per-lookup cost — gate behind operator approval.
- **Timeline**: Bulk ingest can run in hours. Filtering is a SQL query. The bottleneck is outreach quality, not data processing.
- **Storage**: ~5-20K candidate records. Trivial for Postgres.

### Build plan

1. **Download PPP dataset** — CSV bulk download from SBA, filter to target states
2. **Ingest script** — parse CSV, filter by NAICS + loan amount + status, insert into new `PppCandidate` table
3. **Candidate scoring** — derive estimated revenue, match to ontology, compute acquisition fit score
4. **Operator review page** — `/outreach/candidates` — browse, filter, approve for outreach
5. **Outreach draft generator** — personalized letter per candidate using Claude (Haiku)
6. **Manual send** — operator copies/sends via email (automated sending is Phase 2)
