---
name: Spec → plan → subagent-driven execution, branch per feature
description: Validated workflow for substantial Compositor work units — decompose into independent sub-projects, each gets its own brainstorm → spec → plan → branch → PR cycle
type: feedback
originSessionId: b987864d-c0c3-4b30-b74f-2630106ab0da
---
For substantive Compositor work (scaffolding, feature implementation, anything beyond a line-level doc edit), use this pipeline and don't shortcut it:

1. **Brainstorm** via `superpowers:brainstorming` — ask one question at a time, settle on approach, present design in sections. Produces an approved spec at `docs/specs/YYYY-MM-DD-<topic>-design.md`.
2. **Plan** via `superpowers:writing-plans` — the spec turns into `docs/plans/YYYY-MM-DD-<topic>-plan.md` with bite-sized tasks (2-5 min per step, TDD-shaped where applicable, exact commands and code blocks in every step).
3. **Execute** via `superpowers:subagent-driven-development` — fresh subagent per task, two-stage reviews between tasks, mark completed as you go.
4. **Branch per feature** per `COLLABORATION.md` rule 4 — work lives on `<phase>/<feature>` (e.g. `phase-0/scaffold`), never direct to `main`. Push and open a PR via `gh pr create`. User reviews and merges.

**When a work unit is too large for one spec, decompose first.** Compositor's Phase 0 had four bundled subsystems (scaffold, input events, wordlist pipeline, tooling); we split into three sub-projects each with its own brainstorm cycle. Each sub-project should produce working, testable software on its own.

**Why:** Validated by Phase 0 sub-project 1 execution 2026-04-17. The decomposition kept the PR reviewable (57 files on origin, coherent spec → plan → implementation story), the branch-per-feature flow caught a DECISIONS-level divergence mid-execution (SvelteKit amendment) without polluting main, and the subagent-per-task pattern let Haiku handle the mechanical scaffolding while Sonnet took the PWA integration task that needed more judgment.

**How to apply:** When the user asks "let's work on X" and X is non-trivial, default to this pipeline. If X is genuinely small (line-level doc edit, config tweak), direct execution is fine. For anything in between, present the decomposition first and let the user pick the granularity.

**Memory adjacent:** `feedback_brainstorming_handoff.md` covers the "wait for explicit approval before auto-chaining brainstorming → writing-plans" nuance. That applies within this pipeline — after brainstorming produces a spec, pause for user approval before invoking writing-plans.
