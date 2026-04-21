---
name: Brainstorming doesn't auto-chain to writing-plans
description: After a brainstorming session produces a design/sequencing doc, wait for explicit approval before invoking writing-plans — user may want the doc to stand alone
type: feedback
originSessionId: 30eb339b-772c-4689-b85b-9b514d1e25a0
---
After a brainstorming session ends with a committed design or sequencing doc, do NOT automatically invoke the writing-plans skill. Ask first, or wait for the user to request it.

**Why:** On 2026-04-17, after committing `docs/ROADMAP.md` via the brainstorming flow, the user explicitly said "don't write a plan." The brainstorming skill's default terminal state is writing-plans, but sequencing documents (roadmaps, phase plans) are their own deliverable — the doc itself is the output, not a precursor to an implementation plan. The user wanted the roadmap to stand alone until they opt in to planning a specific phase.

**How to apply:** When finishing a brainstorming session, present the committed doc and ask whether to proceed to writing-plans. Treat writing-plans as opt-in, not automatic. This applies especially for planning/sequencing/roadmap docs, but default to asking for *any* brainstorming output rather than chaining automatically.
