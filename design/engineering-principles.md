# Engineering Principles

Applies to all sunj-labs systems — agentic and non-agentic, backend and
frontend. These are the rules that govern how we divide and structure code.

---

## The Core Five

Drawn from SOLID, GoF, and Fowler. Not all five apply to every decision —
use the earn-your-keep test to decide which are in play.

### 1. Separation of Concerns (SoC)

Each module has exactly one reason to change. Mixing concerns — business logic
with persistence, presentation with data fetching, orchestration with business
rules — means unrelated changes break each other and unrelated tests have to
know about each other.

> *A module should be changeable for one reason without touching its
> neighbours.*

The tell: when you find yourself writing "and also" to describe what a module
does, it has two concerns.

---

### 2. Abstraction — implement once, vary at the boundary

Significant logic lives in exactly one place. Callers receive a named
boundary — a function signature, an interface, a type — not a copy of the
implementation. Duplication is the symptom; the cure is an abstraction that
makes the variation explicit.

The DRY principle is the popular formulation, but the deeper point is that
**a boundary makes a contract.** The contract is what callers depend on, not
the implementation behind it. Changing the implementation without changing the
contract is safe. Changing the contract is a breaking change — make it
deliberate.

---

### 3. Dependency Inversion

High-level modules define what they need. Low-level modules implement it.
Never the reverse.

The practical form: design the interface from the caller's perspective, then
write the implementation to satisfy it. The caller owns the contract; the
implementation is replaceable. This is why you write a function signature
before the function body, and an API schema before the handler.

```
Caller defines:    what it needs (the interface)
Implementer owns:  how it's done (hidden behind the interface)
```

The tell: if your high-level business logic imports a specific database driver,
HTTP client, or third-party SDK directly — instead of depending on an
abstraction it controls — the dependency is pointing the wrong way.

---

### 4. Cohesion — keep together what changes together

The equal and opposite force to SoC. SoC says separate what changes for
different reasons. Cohesion says don't separate what changes for the same
reason.

Over-decomposition is as harmful as under-decomposition. A module so thin it
has no gravity — a one-line wrapper, a pass-through, a "util" with a single
function — adds indirection without earning it.

The tell: every feature requires touching six files. Each file is individually
"clean" but the feature has no home.

---

### 5. Open/Closed

Open for extension, closed for modification. When new variation arrives, you
should be able to add it without editing the existing path.

In practice: design extension points where variation is expected. A registry
(new scrapers register themselves), a strategy (new scoring dimensions
configure in), a plugin slot (new agents route without touching the
orchestrator). The existing code doesn't change; the new code slots in.

The tell: adding a new scraper source requires editing a switch statement
in the worker. Adding a new scoring dimension requires adding an `if` branch
to the scorer. Both are Open/Closed violations — the extension point is
missing.

---

## Earn Your Keep

Every separation has a cost: indirection, files to navigate, interfaces to
maintain. A separation earns its keep when it delivers at least one of:

| Benefit | Test |
|---|---|
| **Testability** | Can you verify this module in isolation without mocking its callers? |
| **Replaceability** | Can you swap the implementation without changing callers? |
| **Parallelizability** | Can two developers work on each side simultaneously? |
| **Blast radius reduction** | Does a change here leave unrelated modules untouched? |

If none of these apply, the separation adds cost without benefit. Don't add it.

**The three questions — run before adding any boundary:**

1. What changes independently here? *(SoC trigger — if nothing, don't separate)*
2. What would a test need to know about the caller or infrastructure?
   *(each answer is a dependency that should be injected, not assumed)*
3. How many files break if the implementation changes?
   *(if too many, a facade is missing)*

---

## Where the Principles Apply

These are not agentic-specific. The same rules apply at every layer:

| Layer | SoC boundary | Interface |
|---|---|---|
| UI components | Presentational vs. data-fetching | Props contract |
| API routes | HTTP handling vs. business logic | Request/response schema |
| Business logic | Domain rules vs. persistence | Service function signature |
| Data access | Query construction vs. connection management | Repository interface |
| Workers / agents | Business logic vs. orchestration | Typed input/output |
| External integrations | Client call vs. retry/circuit logic | Wrapper function with owned error types |

The agentic flywheel pattern (in `strategy/sdlc-process.md`) is one
application of this to multi-agent systems. It is not a special case — it
follows the same rules as everything else.

---

## Signs a Boundary is Missing

- A test that needs to set up 4+ things unrelated to what it's testing
- A change to a scraper that requires touching a digest file
- A function that can't be called without starting a database or queue
- Two modules importing each other (circular dependency = missing abstraction)
- A module named `utils.ts`, `helpers.ts`, or `misc.ts`

## Signs a Boundary is Unnecessary

- An interface with exactly one implementation and no plan to add another
- An abstraction layer that just delegates every call without adding anything
- A module that exists to be "organized" but never tested or replaced in isolation
- A facade over something that is already a stable external contract (e.g.,
  wrapping Prisma's `findMany` when Prisma itself is the interface)

---

---

## Canonical References

When you need a pattern for a specific structural problem, look here first
before inventing something new.

| Source | What it covers | Link |
|---|---|---|
| **GoF Design Patterns** (Gamma, Helm, Johnson, Vlissides) | 23 object-oriented patterns for managing dependencies, composition, and behaviour. Creational, structural, behavioural. | https://en.wikipedia.org/wiki/Design_Patterns |
| **Fowler — EAA Catalog** | Patterns for enterprise application architecture: domain logic, data access, ORM, presentation, distribution. The naming layer for backend structure. | https://martinfowler.com/eaaCatalog/ |
| **Fowler — Refactoring** | Named moves for improving structure without changing behaviour. The vocabulary for code review and incremental cleanup. | https://refactoring.com/ |
| **Dijkstra — SoC (1974)** | The original formulation. Short. Worth reading. | https://www.cs.utexas.edu/~EWD/ewd04xx/EWD447.PDF |
| **Hunt & Thomas — The Pragmatic Programmer** | DRY, orthogonality, and the philosophy behind abstraction. | Book |

These are the same relationship as `shadcn/ui` to the frontend stack — not
libraries to install, but the canonical vocabulary and pattern catalogue to
reach for when a structural decision needs a name.
