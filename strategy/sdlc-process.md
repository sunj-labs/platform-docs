# SDLC Flow

## Philosophy

Build less. Ship what matters. Measure outcomes, not output.

Every initiative runs through the same lightweight process. The process exists to force clarity before code, not to create ceremony.

## Lifecycle

```
Thesis (5 min)
  → Shape (30 min, if thesis convinces)
    → Commit (spec + PR/FAQ, if scope warrants)
      → User Stories (work items)
        → Spec (markdown, linked from work item)
          → Spec Review (PR or doc review)
            → Implementation (feature branch)
              → Code Review (PR)
                → CI (automated pipeline)
                  → Deploy
                    → Observe
                      → Retrospect (decision record if architectural)
```

## Rules

1. Nothing gets built without a work item.
2. No work item over size M ships without a spec.
3. Specs are reviewed before code starts.
4. Every merge to main is deployable.
5. Outcomes are measured, not just shipped.

## Cadence

For solo operators or small teams without sprint accountability:

- **Weekly review:** Look at the board. What shipped? What's stuck? What's next?
- **Milestones:** Time-boxed goals. Not deadlines — forcing functions for scope.
- **Decision records:** Written when you make a decision you'll forget in 3 months.

For teams with sprint cadence: map milestones to iterations. The canvas and spec workflow sits upstream of sprint planning.

## Documentation as Byproduct

| Layer | What | When updated |
|-------|------|--------------|
| Decision records | Why decisions were made | When you make a decision |
| Specs | What gets built and why | Before implementation |
| Session logs / stand-ups | What happened | Every session or daily |
| Changelog | What shipped | Every merge to main |
| README | What this is, how to run it | When setup changes |

If you follow the flow, documentation writes itself. If you're writing standalone docs, something is wrong with the process.
