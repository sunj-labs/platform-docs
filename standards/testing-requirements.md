# Testing Philosophy

## Principle

Test the things that would cost you money or time if they broke. Every test must earn its place. No coverage theater.

## Test Pyramid

```
         ╱ ╲
        ╱ E2E ╲        ← Rare. Playwright for critical paths only.
       ╱───────╲
      ╱  Integ  ╲      ← Key integration points: API routes, DB, external services.
     ╱───────────╲
    ╱    Unit     ╲     ← Bulk of tests. Fast, isolated, run on every push.
   ╱───────────────╲
```

## Framework

| Tool | Purpose |
|------|---------|
| **Vitest** | Unit + integration tests. Fast, TypeScript-native, watch mode. |
| **Playwright** | E2E tests. Critical user paths only. |
| **MSW (Mock Service Worker)** | Mock external APIs (Anthropic, Gmail, etc.) in tests. |

## What Gets Tested

### Always test (non-negotiable)

- **Safety rules** — Wrong answer = deleted data, wasted money. 100% branch coverage.
- **Data transformations** — Garbage in/out breaks downstream.
- **Business logic** — The core value of the tool.
- **API contracts** — Breaking change = broken system.

### Test when practical

- Input validation — bad UX on wrong input
- Error handling paths — graceful degradation matters
- Configuration loading — wrong config = silent failures

### Don't test

- Third-party library internals — that's their job
- Trivial getters/setters — no logic, no risk
- LLM output content — non-deterministic; use Langfuse evals instead
- UI layout — visual regression is overkill at small scale

## Coverage Policy

No hard coverage floor. Coverage percentage is a vanity metric. Instead:

- Safety rules: 100% branch coverage. No exceptions.
- Business logic: ≥80% line coverage. Test the branches that matter.
- Glue code / config: test if it's broken you before, skip if it hasn't.

## Naming Convention

```
describe('dealScorer', () => {
  it('returns zero when revenue is missing', () => { ... });
  it('scores high-margin deals above low-margin', () => { ... });
  it('throws on negative asking price', () => { ... });
});
```

Pattern: `describe(unit)` → `it(behavior)`. Readable without comments.

## Fixture Pattern

- Shared fixtures and factories in `tests/fixtures/`
- Mock external APIs with MSW — don't test what the API returns, test that you called it correctly, handled the response shape, and enforced your rules on the output
- Factory functions over hardcoded test data:

```typescript
function makeDeal(overrides?: Partial<Deal>): Deal {
  return {
    name: 'Acme Testing Labs',
    revenue: 1_200_000,
    profit: 380_000,
    askingPrice: 1_500_000,
    source: 'bizbuysell',
    ...overrides,
  };
}
```

## CI Integration

```yaml
- name: Test
  run: npx vitest run --coverage

- name: E2E (critical paths only)
  run: npx playwright test
```

- Run unit + integration tests on every push
- Report coverage to CI log, don't fail on percentage
- Fail on test failures only
- E2E tests run on PRs to main, not every push
- Integration tests excluded from coverage metrics (they depend on external state)
