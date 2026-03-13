# Memory & Session Continuity

## The Real Memory Problem

The problem isn't architectural — it's operational. Context gets lost between
sessions with AI assistants, between agent runs, and between work days. The
fix is discipline + structure, not new infrastructure.

## Three Layers of Memory

1. **Git is the ground truth.** Specs, ADRs, canvases, CHANGELOG.md — these
   are durable, versioned, searchable. If a decision matters, it's in git.

2. **CLAUDE.md per repo.** A file at the root of each repo that tells any AI
   assistant the project context, conventions, current state, and gotchas.
   Every session starts by reading it.

3. **Structured session logs.** Timestamped records of every working session
   with frontmatter that captures state transitions.

## Session Log Template

Lives in each repo at `/sessions/`:

```markdown
---
session_id: 2026-02-28-1430
project: {project-name}
agent: personal
entry_time: 2026-02-28T14:30:00-08:00
exit_time: 2026-02-28T16:45:00-08:00
status: completed
tags: [ci-cd, github-actions, deployment]
---

# Session: {Title}

## Entry State
- What was the state before this session?

## Work Done
- What was accomplished?

## Exit State
- What is the state now?

## Decisions Made
- Key decisions and their rationale

## Open Threads
- [ ] Unfinished items

## Next Session Should Start With
Read CLAUDE.md. Review the open threads above.
```

## CLAUDE.md Template

```markdown
# CLAUDE.md — [Project Name]

## What This Is
[One sentence description]

## Current State
[What's deployed, what's in progress, what's blocked]

## Architecture
[Key components, how they connect, where they run]

## Conventions
- Branch naming: feature/ISSUE-NNN-description
- Commit messages: conventional commits (feat:, fix:, docs:)
- Tests: pytest, run with `make test`
- Deploy: push to main triggers CI

## Known Gotchas
- [Thing that will bite you if you don't know about it]

## Recent Changes
- [Date]: [What changed and why]

## Key Files
- /specs/ — Feature specifications
- /sessions/ — Session logs with structured frontmatter
- /src/ — Source code
- CHANGELOG.md — Release history
```

## The Rule

If you follow the SDLC flow (canvas → spec → session log → CHANGELOG),
documentation writes itself. If you find yourself writing a standalone
"documentation doc," something is wrong with the process.
