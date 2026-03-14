---
session_id: 2026-03-14-poa-genesis
date: 2026-03-14
duration: ~4 hours
repos: [sunj-labs/poa, sunj-labs/platform-docs]
tags: [agentic-systems, sdlc-evolution, deal-sourcing, distributed-family, power-of-attorney, financial-modeling]
skills: [product-canvas, pr-faq, system-design, uml-diagrams, prisma-schema, nextjs, agent-architecture, sdlc-design]
---

# Session: POA Genesis — From Interview to Running App in One Session

## What Shipped

- **sunj-labs/poa repo** created (private, GitHub)
- **Product Canvas** — 3-stage progressive canvas for the POA platform
- **PR/FAQ** — Amazon-style press release crystallizing the value prop
- **4 Phase 1 Specs** — Deal Sourcing, Scoring Pipeline, Dashboard/Deals UI, Notifications
- **11 Design Diagrams** (all Mermaid):
  - ERD (22 models), Use Case (4 roles inc. ADVISOR), Component (7 agents + curator), Deployment
  - State: Deal lifecycle with agent authorization contract
  - Sequences: deal sourcing flow, consensus flow, outreach flow
  - Activities: scoring pipeline, digest compilation
- **Dexter integration mapping** — how virattt/dexter patterns adapt to private business acquisition
- **Full app scaffold** — Next.js 16, Prisma, BullMQ (5 queues), NextAuth/Google OAuth, Tailwind, health check passing
- **Prisma schema** — 22 models including DealList (shortlists), CurationRecommendation, Proforma, ProformaAssumption, LocationProfile
- **5 Phase 1 Epics** in GitHub with stories and acceptance criteria
- **SDLC standards updated** — Design step added between Canvas and Specs, state diagrams elevated to first-class for agentic systems

## What Was Learned

- **State diagrams are specifications in agentic systems, not documentation.** They define what agents are allowed to do — which transitions, which require human approval, what the recovery paths are. Without them, agents do whatever they want.
- **The SDLC was missing a Design step.** Going from Canvas/PR-FAQ straight to Specs skipped the structural modeling that reduces ambiguity. The full funnel is now: Interview → Canvas → PR/FAQ → Design Diagrams → Specs → Epics → Build. Each step drives increasing certainty.
- **Shortlists are the collaboration surface, not individual deals.** The family doesn't need to see every scored deal — they need curated collections shared with the right people (including external advisors like CPA, wife).
- **Curation is a distinct agent function from scoring.** Scoring is mechanical (apply filters + rubric). Curation is strategic (portfolio fit, timing, complementary businesses, tax geography).
- **Location is not a flat attribute.** Tax geography (state income tax, estate tax, business-friendly ranking) is a first-class scoring and proforma dimension.
- **Dexter is a reference architecture, not a runtime dependency.** Its plan→execute→verify loop and structured financial output patterns adapt to private business analysis, but its data sources (SEC filings, stock prices) don't apply.

## SDLC Evolution

- Added **Interview** as an explicit first step (was implicit)
- Added **Design** step between Canvas/PR-FAQ and Specs (7 diagram types with guidance on when each earns its keep)
- **State diagrams** elevated to REQUIRED for any agentic system — they define the agent authorization contract
- **Object model** must be updated before specs are written (prevents naming drift)
- Rule added: "Design diagrams are produced before specs for any initiative with async flows or multiple components"

## Skills Demonstrated

- **Product thinking**: Translated a complex family situation (POA, stroke care, property sale, business acquisition) into a structured product canvas and PR/FAQ
- **Interactive requirements discovery**: 6 rounds of structured interview, progressively narrowing from "what's the problem" to "what are the hard filters on deal scoring"
- **Distributed systems design**: 7-agent architecture with 5 BullMQ queues, event-driven scoring, subprocess execution with trust boundaries
- **Financial domain modeling**: SBA 7(a) rules, 1031 exchange constraints, EBITDA multiples, proforma projections, tax geography
- **SDLC design**: Identified and filled gaps in an existing lightweight SDLC process, adding rigor without ceremony
- **UML/design diagramming**: Full suite of 7 diagram types, all in Mermaid, all serving specific design decisions

## Collaboration Model

Human drove the "what" and "why" — the family situation, financial constraints, deal criteria, operating model preferences. AI drove the "how" — system architecture, schema design, agent patterns, SDLC process improvements. Human caught several gaps the AI missed:
- SDLC should come before implementation plans
- Design diagrams were missing from the SDLC
- State diagrams are first-class in agentic systems
- Collaborator model was too flat (needed ADVISOR + scoped DealLists)
- Curation is a distinct agent function
- Proformas and tax geography are first-class concerns
- Dexter's concrete integration points needed explicit mapping

The pattern: AI proposes, human stress-tests and identifies structural gaps, AI integrates and builds.

---

## Post Ideas

### 1. "State Diagrams Are Specifications, Not Documentation — Lessons from Building an Agentic System"

When your system has autonomous agents making decisions, a state diagram isn't a nice-to-have artifact you draw after the code works. It's the contract that defines what each agent is *allowed* to do. Building a deal-sourcing platform with 7 AI agents taught me that the most important design artifact wasn't the sequence diagram or the component diagram — it was the state machine that said "the Scorer can only touch NEW deals, the Curator can recommend but never transition, and nothing moves past SCORED without a human." Without that, you don't have an agentic system — you have chaos with an API key.

### 2. "From Phone Calls to Pipeline: Building an Async-First Platform for a Family Scattered Across 4 Time Zones"

My father had a major stroke 18 months ago. My parents need $700K/year in round-the-clock care. We're selling a property and buying a business to fund it — but I'm in the UK, my sister's in Wilmington, my brother's in Seattle, and my mom is the primary caregiver with zero bandwidth for deal sourcing. I built an agentic platform that scrapes business listings, scores them against AI-resistance and necessity criteria, and surfaces a daily digest to each family member in their timezone. Mom gets a calm email at 8am. I get a ranked pipeline. Nobody needs to be on the phone at 2am. The system works while we sleep.

### 3. "Your SDLC Is Missing a Step: Why Design Diagrams Belong Between Requirements and Implementation"

Most lightweight SDLC processes go straight from "what to build" (spec) to "let's build it" (tickets). That works for CRUD apps. It fails for agentic systems with async queues, multiple autonomous agents, and event-driven pipelines. I added a Design step — ERD, component diagram, sequence diagrams, activity diagrams, state machines — between Canvas/PR-FAQ and Specs. It took one session to realize that writing specs without modeling the agent interactions first was like writing user stories without knowing who the users are. The diagrams aren't ceremony — they're the thing that turns "I think I know how this works" into "I can prove how this works."
