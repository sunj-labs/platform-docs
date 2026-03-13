# Security Scanning Standard

Three layers. All run in CI via GitHub Actions, blocking merge on failure.

## Pipeline Order

```
push → secret scan → lint → typecheck → SAST → dependency audit → test → build → deploy
       ^^^^^^^^^^^^                      ^^^^   ^^^^^^^^^^^^^^^^^
       Fail fast on                      Catch  Fail on known
       leaked creds                      vulns  vulnerabilities
```

## Layer 1: Secret Detection (pre-commit + CI)

**Tool:** gitleaks

**What it catches:** API keys, OAuth tokens, AWS credentials, passwords, private keys.

**Critical for:** Anthropic API keys, NextAuth secrets, OAuth tokens, AWS credentials.

**CI config:**
```yaml
- name: Detect secrets
  uses: gitleaks/gitleaks-action@v2
```

**Local pre-commit hook** (so secrets never reach the remote):

Option A — husky (Node-native):
```bash
# .husky/pre-commit
npx gitleaks detect --source . --verbose
```

Option B — pre-commit framework:
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

Install: `npm pkg set scripts.prepare="husky"` (Option A) or `pip install pre-commit && pre-commit install` (Option B).

## Layer 2: Dependency Vulnerabilities (CI)

**Tool:** npm audit

**What it catches:** Known CVEs in installed packages.

```yaml
- name: Dependency audit
  run: npm audit --audit-level=high
```

For stricter enforcement, use `better-npm-audit` or `audit-ci`:
```yaml
- name: Dependency audit
  run: npx audit-ci --high
```

## Layer 3: Static Analysis / SAST (CI)

**Tool:** ESLint security plugins + TypeScript compiler

**What it catches:** Injection risks, unsafe patterns, type errors.

```yaml
- name: Type check
  run: npx tsc --noEmit

- name: Lint + security rules
  run: npx eslint . --max-warnings 0
```

ESLint security plugins to include:
- `eslint-plugin-security` — detects unsafe regex, eval, non-literal require
- `@typescript-eslint/eslint-plugin` — type-aware linting rules
- `eslint-plugin-no-secrets` — catches hardcoded secrets in source

For deeper SAST (optional, as complexity grows):
- **Semgrep** — language-aware static analysis, free tier, TypeScript support
```yaml
- name: SAST
  uses: returntocorp/semgrep-action@v1
  with:
    config: p/typescript
```

## Linting (separate from security, same pipeline)

**Tool:** ESLint + Prettier

```yaml
- name: Lint
  run: npx eslint .

- name: Format check
  run: npx prettier --check .
```

## Adding to a New Repo

Each repo calls the org-level reusable workflow:

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: sunj-labs/.github/.github/workflows/ts-ci.yml@main
```

That's it. The reusable workflow handles the full pipeline.
