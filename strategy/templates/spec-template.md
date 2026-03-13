# SPEC-NNN: [Feature Name]

## Canvas Reference
Link to product canvas or PR/FAQ.

## Context
Why now? What triggered this work?

## Requirements
### Must Have
- ...
### Should Have
- ...
### Won't Have (this version)
- ...

## Design
### User Flow
Step-by-step, what the user (or system) does.

### Object Model
What are the core objects? How do they relate?
(Define objects, not screens.)

### Interface
Wireframe, CLI spec, API contract — whatever's appropriate.

## Technical Approach
Architecture, dependencies, risks.

## Deployment & Access Architecture

Answer these before writing code. The answers determine your deployment tier,
networking, auth, and domain strategy.

### Who accesses this?

| Question | Answer |
|----------|--------|
| Is this operator/team-only (internal)? | yes / no |
| Will trusted external people use it (partners, clients)? | yes / no |
| Will the public use it (customers, strangers)? | yes / no |
| Does access need to work from outside the corporate network / VPN? | yes / no |

### What does it need?

| Question | Answer |
|----------|--------|
| Does it persist data (database, filesystem)? | yes / no |
| Does it handle secrets server-side (API keys, OAuth tokens)? | yes / no |
| Does it need SSR or API routes? | yes / no |
| Does it call other internal services? | yes / no |
| Does it need auth (login, user accounts, SSO)? | yes / no |

### Deployment decision

Based on the answers above:

| Condition | Tier |
|-----------|------|
| Internal-only, no persistence, validating UI | **Mockup** (prototype, local dev, design tool) |
| Needs a shareable link, still mock data, no backend | **Staging** (static hosting, internal CDN) |
| Any "yes" to persistence, secrets, SSR, or service calls | **Production** (full deployment to your infra) |
| Public users outside corporate network | **Production + edge** (CDN, TLS, public domain) |

### If public-facing, additionally answer:

| Question | Answer |
|----------|--------|
| Domain name? | |
| TLS strategy? | |
| Auth provider? (OAuth, SSO, magic link, none?) | |
| Rate limiting needed? | yes / no |
| Do we need a public status page? | yes / no |
| GDPR / data handling implications? | |

### Monitoring

| Question | Answer |
|----------|--------|
| Healthcheck endpoint path? | `/health` or specify |
| Add to infrastructure monitoring? | yes / no |
| LLM/AI observability needed? | yes / no |
| Cost controls relevant? | yes / no |

## Acceptance Criteria
How we know it's done. Testable statements.

## Open Questions
Things we don't know yet.
