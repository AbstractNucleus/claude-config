---
name: Git analysis summary format
description: Preferred shape when user asks "analyze git" or similar — concise grouped-by-PR summary with branch state and untracked files
type: feedback
originSessionId: 69384601-1448-493b-9f3e-e2880d3fabe0
---
When asked to "analyze git" or for a git summary, produce a compact write-up grouped by logical unit (PR/feature) rather than a flat commit list. Include: what landed, rough timing, current branch state, and any untracked files. Flag commits authored by anyone other than the user. End with a short clarifying question if the scope is ambiguous.

**Why:** User validated this exact format on 2026-04-18 after session-start git summary — said "Yes, exactly. I wanted that type of summary." Flat `git log` dumps are noise; grouped-by-PR narrative is the signal.

**How to apply:** Default to this shape whenever the user asks for git status/history/activity analysis. Scale length to volume — one line for trivial activity, a short paragraph for substantial work.
