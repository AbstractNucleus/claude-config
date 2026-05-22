---
name: improve
description: Generate an improvement, addition, or fix for the current project, propose it once, then either grill the user on it or implement it. Use when the user invokes /improve, asks "what's next", "polish this", or wants the project moved forward. With an argument (e.g. /improve add dark mode), researches and implements that specific idea instead of generating one.
---

# /improve

One proposal per invocation. Then commit (with grilling) or go.

## Mode A — no argument: pick something worth doing

1. **Scan the project.** In parallel where possible:
   - `README.md`, `CLAUDE.md`, `docs/` — stated intent and constraints.
   - `git log -20 --oneline` and `git status` — what's in flight, what's dirty.
   - `gh issue list --limit 20` (only if `gh` is available and the repo is on GitHub).
   - Grep for `TODO|FIXME|XXX` — known cruft the author already flagged.
   - Smell pass: missing tests for recent code, broken CI, dead code, repeated patterns, stale config, missing error paths at real boundaries (not internal ones).
2. **Generate 5–10 candidate ideas internally.** Do not show the list.
3. **Pick one** that is high-value and small enough to land in this session. If 2–3 candidates are tightly related (same flow, same file, same bug class), bundle them — otherwise ship a single one.

## Mode B — argument given: the user has the idea

`/improve <idea>` — research the idea, then propose how you'd ship it.

1. Read the code paths the idea touches. Look at how similar features are already done in this project.
2. If it's a library/framework choice or an unfamiliar pattern, do one focused web search for current best practice. Don't go deeper unless the user is around.

## The proposal (both modes)

Write ONE message with this exact shape, then end the turn:

```
**What** — <1–3 sentences, concrete change>
**Why** — <user-facing impact, debt paid, or risk reduced>
**Scope** — <files likely touched, rough size>
**Verify** — <the one command the user will run>
```

Do not ask "shall I proceed?". The next user message decides:
- "grill me" (or similar) → drop into a stress-testing interview about the proposal before writing any code.
- anything else, or no specific direction → implement.

## Implementing

Before writing code, invoke `karpathy-guidelines` via the Skill tool. Then check for project-specific standards: `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, `docs/style*`. Honor both layers.

Surgical changes only. No drive-by refactors. No speculative abstractions. No features outside the proposal's scope. If you discover related work along the way, use the spawn-task tool to flag it rather than expanding the diff.

## Self-review before reporting done

Read your own diff (`git diff`) and check:

- [ ] Karpathy guidelines respected (surgical, no overcomplication, assumptions surfaced).
- [ ] Project coding standards respected.
- [ ] Verify command actually runs and shows the change. Run it.
- [ ] Comments explain WHY only, never WHAT. None that reference "this PR" or "added for X".
- [ ] No dead code, half-finished bits, or unused imports.
- [ ] No backwards-compat shims for code that doesn't exist yet.

Fix anything that fails before reporting.

## Done

End with:

> Done. Run `<verify command>` to see it.

Nothing more.
