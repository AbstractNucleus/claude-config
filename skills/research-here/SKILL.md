---
name: research-here
description: Use when the user wants structured research written into the current project, invoked via "/research-here [topic]", "research this", "deep dive in this repo", or any unqualified research request. If no topic is given, infers it from the current branch, recent commits, in-progress files, and TODOs. Decomposes into parallel angles, verifies URL-per-claim citations via WebFetch, optionally challenges with a skeptic, and writes to ./research/.
argument-hint: [shallow] [topic]
---

# Research Here

## Overview

Project-scoped research pipeline. Writes plain markdown with inline URL citations and a Sources table into `./research/` in the current working directory.

Default mode (no topic given): infer the topic from the project's current work, the current branch name, recent commits, in-progress changes, open TODOs, or any active issue context. Use this when the user just says "research this" without specifying what.

## When to Use

- User types `/research-here [topic]` from inside a project
- User says "research this", "research this here", "deep dive in this repo", "look into this"
- User asks for research without specifying the destination, this is the default research skill
- Out-of-scope: quick factual lookups; capturing user's own thoughts as a note.

## Argument Parsing

`argument-hint: [shallow] [topic]`

- `shallow <topic>`, skip the skeptic + re-research loop.
- `<topic>`, deep mode. Include skeptic + re-research on `high`/`medium` challenges.
- (no topic), deep mode, topic inferred from current project work.

If the first positional arg is exactly `shallow` AND the remaining string is non-empty, treat the rest as the topic. If the arg is just `shallow` with nothing after, treat the whole arg as the topic and use deep mode (don't strip "shallow" from a topic like "shallow water rendering"). Otherwise the whole arg string is the topic. If empty, infer from project state (see step 1).

## Safety Posture

- If cwd has no git repo, still run, just skip the branch/push/PR steps and report local paths at the end. (Topic inference falls back to in-progress files + TODOs when there's no git history.)
- Create the output folder even if it means creating `./research/`. Do not ask first, it's a documentation folder, creation is safe.
- Do NOT modify any code in the repo. Only write files inside `./research/YYYY-MM-DD-<topic-slug>/`.
- Expect a full deep run to take several minutes: dispatching 5–7 parallel Researchers plus a Verifier that WebFetches every URL is slow. Don't abort on slow sub-agents.

## Workflow

### 1. Parse args + preflight

- Extract `depth ∈ {shallow, deep}` (default `deep`) and `topic`.
- **If `topic` is empty, infer it from the project's current work**:
  - Read current branch name (`git branch --show-current`), strip `feat/`, `fix/`, `wip/` prefixes; if descriptive, use it.
  - Read last 5 commit subjects (`git log -5 --format=%s`), look for a coherent theme.
  - Check `git status` for in-progress files; scan them and `git diff` for what's being changed.
  - Look for TODO/FIXME/XXX comments in modified files, and any open issue tag in the branch name.
  - Pick the most specific signal, branch name beats commit log beats diff scan. Surface the inferred topic to the user in one sentence before proceeding ("Inferred topic: <topic>. Researching now."). Do not block on confirmation, auto mode.
  - If nothing usable surfaces (clean tree, generic branch, no TODOs), ask once: "I couldn't infer a topic from the project state. What do you want researched?"
- Compute `topic_slug` (kebab-case ≤ 60 chars), `today = YYYY-MM-DD`, `stamp = YYYY-MM-DD-HHMM`.
- Check `git rev-parse --is-inside-work-tree` to decide whether PR steps apply.
- Compute `out_dir = <cwd>/research/{today}-{topic_slug}`.

### 2. Decompose

- Read `agent-prompts.md` from this skill's folder, §Decomposer.
- Dispatch one Agent call with the Decomposer prompt and user's topic.
- Validate: 5–7 angles, non-empty `boundaries`.

### 3. Research (parallel)

- Read `agent-prompts.md` from this skill's folder, §Researcher.
- Dispatch ONE Agent call per angle, all in a single message (parallel).
- Collect outputs by angle id.

### 4. Verify

- Read `agent-prompts.md` from this skill's folder, §Verifier and `verification-rubric.md`.
- Dispatch one Verifier Agent call. Parse JSON.
- Check global abort conditions, if triggered, stop and report. Do not synthesize.

### 5. Skeptic (deep only)

- If `depth == shallow`, skip to step 7.
- Read `agent-prompts.md` from this skill's folder, §Skeptic.
- Dispatch one Skeptic Agent call. Parse challenges JSON.

### 6. Re-research (deep only)

- For each `high`-severity challenge, dispatch a Re-researcher in parallel.
- For `medium`, dispatch in parallel with synthesis scaffolding.
- Apply the skeptic-handling matrix from the rubric.

### 7. Synthesize

- Read `agent-prompts.md` from this skill's folder, §Synthesizer, local mode.
- Filter claims: only those with an OK verdict that survived the Skeptic (in deep mode).
- Dispatch the Synthesizer Agent. It writes a `README.md`-shaped document with inline URLs and a Sources table.

### 8. Write files

Write to `{out_dir}`:

```
{out_dir}/
  README.md                                ← synthesis (local-mode shape)
  angles/
    a1-{angle1_slug}.md                    ← raw researcher output with URLs
    a2-{angle2_slug}.md
    ...
  verification.json                        ← verifier JSON (+ skeptic if deep)
  sources.md                               ← flat table of every URL + verdict + authority
```

`README.md` is the user-facing artifact. `angles/` is the per-angle raw, `verification.json` is the audit trail, `sources.md` is the deduplicated URL ledger for quick scanning.

### 9. Git + PR (if cwd is a git repo)

```bash
git checkout -b research/{topic_slug}-{today}
git add research/{today}-{topic_slug}/
git commit -m "research: {topic}"
git push -u origin research/{topic_slug}-{today}
gh pr create --title "research: {topic}" --body "<summary>"
```

If `gh` isn't installed or authenticated, stop after the push and tell the user the branch is ready for them to open the PR manually.

PR body:

```markdown
## Summary
- Topic: {topic}
- Depth: {depth}
- Angles: {N}
- Verifier: {ok}/{total} OK, {weak} WEAK, {dead} DEAD, {hallucinated} HALLUCINATED
- Skeptic (if deep): {overall_assessment}, {N} challenges, {N} resolved

## Files
- research/{today}-{topic_slug}/README.md
- research/{today}-{topic_slug}/angles/ … (list)

## Open questions
- <from skeptic, if any>
```

If cwd is not a git repo, skip 9. Just report paths in step 10.

### 10. Report

Tell the user:
- Output directory path
- PR URL (if a PR was created)
- Verifier summary
- Skeptic verdict if deep
- Any angles that failed verification, never silently dropped

## Partial Failure Handling

Follow `verification-rubric.md` exactly:
- Per-claim failure matrix
- Skeptic-challenge matrix
- Global abort conditions
- Non-aborting degradation (add `## Angles we could not verify` section to README.md)

## Common Mistakes

- **Asking for a topic when one can be inferred.** If the branch name, recent commits, or in-progress diff makes the topic obvious, just announce the inferred topic and proceed. Only ask when project state genuinely surfaces nothing.
- **Modifying repo code.** This skill only writes under `./research/`. Never edit source files; if research surfaces a fix, mention it in the README and stop there.
- **Creating a PR against `main` without a branch.** Always cut a `research/...` branch first.
- **Skipping the PR step when cwd is a git repo but the remote is missing.** Commit and push to local branch anyway; tell the user the PR step needs a remote.
- **Silently dropping failed angles.** Always surface them in `## Angles we could not verify`.
