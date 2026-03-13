# Key References

## Source Attribution

| Source | Key Takeaway | Applied Where |
|--------|-------------|---------------|
| **Anthropic — Boris Cherny's 3 Principles** | Intent > input. Honesty about limits. Respect autonomy. | Design Principles #2, #4, #7 |
| **Anthropic — Catherine Wu / Process** | Prototype first. Build for future models. Underfund on purpose. | SDLC cadence, solo operator philosophy |
| **Anthropic — Constitutional AI** | Explicit, auditable values. Make the system's principles visible. | "Show the work" principle |
| **Google — Titans Architecture** | Surprise-based memory. Short-term + long-term + persistent. MAC variant. | ADR-003, memory layer |
| **lucidrains/titans-pytorch** | MIT license, 1.9k stars, actively maintained. Upgrade path for Option C. | Future Titans model training |
| Intercom Full-Stack Design System | Design objects, not screens. Shared model from concept to code. | Object model, shared language |
| NNG — AI for UX | AI as assistant, not replacement. Human judgment curates AI output. | Conservative automation principle |
| NNG — Future-Proof Designer | Strategy > pixels. Storytelling. Systems thinking. Generalist advantage. | Canvas/PR-FAQ before code |
| Amazon PR/FAQ | Write the press release first. If it doesn't excite, don't build. | PR/FAQ template |
| Lean Product Canvas | Force clarity in 30 minutes or less. | Product canvas template |
| IDEO Human-Centered Design | Respect the user. Design for real contexts. | Design Principle #7 |
| Apple HIG | Progressive disclosure. Clarity. Restraint. | Design Principle #5 |
| Google Material | Systematic visual language. Tokens over ad-hoc decisions. | Design tokens |

## Anthropic Process Principles (Catherine Wu / Boris Cherny)

How Anthropic actually builds products — and why it matters:

**Prototype first.** Skip the spec when you can build a working version faster. The prototype becomes the spec. Internal usage becomes the research. Feedback becomes the roadmap.

**Build for the model six months from now.** Don't optimize for current limitations. Build the architecture that will work when the models improve.

**Underfund on purpose.** Small teams (or solo operators) forced to rely on AI tools ship faster than large teams doing it manually.

**Everyone codes.** PMs, designers, finance, data scientists — everyone writes code. Roles blur. The generalist wins.
