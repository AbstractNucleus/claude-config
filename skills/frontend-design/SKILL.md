---
name: frontend-design
description: Build distinctive, production-grade frontend interfaces. Use when the user asks to build, design, or refactor web components, pages, or applications, including styling, layout, interaction, animation, or any UI-shaped change. Bias toward original, polished output that avoids generic AI-template aesthetics.
---

# Frontend Design

Starter skill for UI work. Owned and edited by you. Develop it as you ship.

## Setup

This skill reads `PREFERENCES.md` in its folder for your default stack and design tastes. The file is tracked in git (shared across machines) but is *yours*, edit freely.

If `PREFERENCES.md` is missing or empty, fall back to the inline defaults in [Working principles](#working-principles) below. You may also ask the user once at task start: "Anything specific I should know about stack or style before I start?", but only on substantial UI tasks, not small tweaks.

## When to Use

- User asks to build a component, page, screen, or full app
- User asks to restyle, redesign, or refactor existing UI
- User asks for animation, interaction, micro-interactions
- User asks "make this look better" or critiques an existing screen
- Any task that produces user-visible markup, styles, or interactive behavior

## Working principles

These apply unless `PREFERENCES.md` overrides them.

1. **Distinctive over generic.** Avoid the default-AI look, generic gradients, centered cards, hero + 3-feature-grid layouts. If your first instinct is "Tailwind defaults on a white background," sit with the brief longer and find a specific point of view.
2. **Show, don't promise.** When the user describes a feature, sketch the actual interaction, what happens on hover, click, scroll, error. Don't ship dead chrome.
3. **Real content over lorem ipsum.** Use plausible copy, plausible numbers, plausible names. Generic placeholders kill the impression of a real product.
4. **Constraints from the brief.** If the user mentions a brand, a domain, a vibe, a reference, pull those into actual decisions (palette, type, density, motion). Don't ignore them.
5. **Verify in a browser** when the change is non-trivial. Tests don't catch visual regressions. See if it runs, click through the flow, look at edge cases.

## Workflow

1. Read `PREFERENCES.md` (if present).
2. Read project-local style files (`CLAUDE.md`, `AGENTS.md`, design tokens, existing components), match what's there before introducing new patterns.
3. For substantial work, propose direction in 3-5 sentences before writing code. Get a nod.
4. Implement with the principles above.
5. Verify in a browser when feasible.

## Notes

- This is a starter. Replace these principles with your real opinions as you find them.
- For one-off questions ("how do I center a div"), skip the workflow, just answer.
