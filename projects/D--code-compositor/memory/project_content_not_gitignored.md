---
name: content/ is never gitignored
description: In Compositor, everything under content/ (including generated word frequency tables) is committed; nothing in content/ should ever be added to .gitignore
type: project
originSessionId: f2683020-5177-42b2-a3db-a12d1f0201d8
---
Nothing under `content/` in the Compositor repo is supposed to be gitignored. That includes generated data like `content/Word Frequency/**` — those JSON tables are checked in, not regenerated per-clone.

**Why:** User stated this directly on 2026-04-17 while reviewing PR #9 (word frequency pipeline). The PR briefly gitignored `content/Word Frequency/` in commit `d9dba0e` and then reversed it in `19cd5bb` — the reversal was correct, the initial add was the mistake. Likely downstream reason: the `getRarity()` TypeScript module and the Tauri app need these tables at runtime without requiring a Python toolchain in the app build.

**How to apply:**
- Never suggest adding anything under `content/` to `.gitignore`.
- When reviewing PRs, treat committed generated data under `content/` as intended, not as a smell.
- If a plan proposes gitignoring anything under `content/`, push back before it lands.
- This is a *project policy*, not yet formally in `docs/DECISIONS.md`. If you touch `DECISIONS.md`, consider proposing an entry capturing this.
