---
name: karpathy-guidelines
description: Behavioral guidelines to reduce common LLM coding mistakes. Use at the start of non-trivial coding tasks (new features, refactors, multi-file changes, bug fixes with unclear root cause) — not for one-line edits or mechanical changes.
---

# Karpathy Guidelines

Behavioral guidelines to reduce common LLM coding mistakes, derived from [Andrej Karpathy's observations](https://x.com/karpathy/status/2015883857489522876) on LLM coding pitfalls.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If something is unclear, stop and name what's confusing before asking.
- If multiple interpretations exist, present them — don't pick silently.
- If the user's approach has a known pitfall, name it before implementing.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Before finalizing: roughly count the lines you added. If it's more than ~2x what the task obviously needs, delete and retry.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it, don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For tasks with 3+ distinct steps, state a brief plan with a verify check per step. Skip the plan for two-step tasks — it's ceremony.

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.
