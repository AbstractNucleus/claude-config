---
name: research-here
description: Use when the user wants structured research written into the current working directory rather than their Obsidian vault — invoked via "/research-here <topic>", "research this here", "do a deep dive in this repo". Decomposes into 5–7 parallel angles, dispatches researcher subagents with strict URL-per-claim citations, verifies every URL by WebFetch, runs a skeptic to challenge weak claims, and synthesizes into ./research/YYYY-MM-DD-<topic>/. Opens a PR if cwd is a git repo.
argument-hint: [shallow] <topic>
---

# Research Here — Local Mode

## Overview

Same research pipeline as `/deep-research`, but writes project documentation into the current working directory instead of the user's vault. Output is plain markdown with inline URL citations and a Sources table — not vault-template pages with wikilinks.

Use this when the research is project-scoped (belongs with the code) rather than wiki-scoped (general knowledge).

## When to Use

- User types `/research-here <topic>` from inside a project
- User says "research this here", "research this for the project", "deep dive in this repo"
- User explicitly does NOT want output in their vault
- Out-of-scope: wiki-scoped research (use `/deep-research`), quick factual lookups (use `/query`), own-thought capture (use `/note`)

## Argument Parsing

`argument-hint: [shallow] <topic>`

- `shallow <topic>` — skip the skeptic + re-research loop.
- `<topic>` — default (deep). Include skeptic + re-research on `high`/`medium` challenges.

If the first positional arg is exactly `shallow`, treat the rest as the topic. Otherwise the whole arg string is the topic.

## Safety Posture

- If cwd is the vault directory (`C:\Users\noelh\Documents\vault`), **refuse**. Suggest `/deep-research` instead. Do not write vault-style pages through this skill.
- If cwd has no git repo, still run — just skip the branch/push/PR steps and report local paths at the end.
- Create the output folder even if it means creating `./research/`. Do not ask first — it's a documentation folder, creation is safe.
- Do NOT modify any code in the repo. Only write files inside `./research/YYYY-MM-DD-<topic-slug>/`.

## Workflow

### 1. Parse args + preflight

- Extract `depth ∈ {shallow, deep}` (default `deep`) and `topic`.
- If `topic` is empty, ask once: "What topic do you want researched?"
- Compute `topic_slug` (kebab-case ≤ 60 chars), `today = YYYY-MM-DD`, `stamp = YYYY-MM-DD-HHMM`.
- Resolve cwd. If cwd is the vault path, refuse and suggest `/deep-research`.
- Check `git rev-parse --is-inside-work-tree` to decide whether PR steps apply.
- Compute `out_dir = <cwd>/research/{today}-{topic_slug}`.

### 2. Decompose

- Read `../\_research-shared/agent-prompts.md` §Decomposer.
- Dispatch one Agent call with the Decomposer prompt and user's topic.
- Validate: 5–7 angles, non-empty `boundaries`.

### 3. Research (parallel)

- Read `../\_research-shared/agent-prompts.md` §Researcher.
- Dispatch ONE Agent call per angle, all in a single message (parallel).
- Collect outputs by angle id.

### 4. Verify

- Read `../\_research-shared/agent-prompts.md` §Verifier and `../\_research-shared/verification-rubric.md`.
- Dispatch one Verifier Agent call. Parse JSON.
- Check global abort conditions — if triggered, stop and report. Do not synthesize.

### 5. Skeptic (deep only)

- If `depth == shallow`, skip to step 7.
- Read `../\_research-shared/agent-prompts.md` §Skeptic.
- Dispatch one Skeptic Agent call. Parse challenges JSON.

### 6. Re-research (deep only)

- For each `high`-severity challenge, dispatch a Re-researcher in parallel.
- For `medium`, dispatch in parallel with synthesis scaffolding.
- Apply the skeptic-handling matrix from the rubric.

### 7. Synthesize

- Read `../\_research-shared/agent-prompts.md` §Synthesizer — local mode.
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
- Any angles that failed verification — never silently dropped

## Partial Failure Handling

Follow `../\_research-shared/verification-rubric.md` exactly:
- Per-claim failure matrix
- Skeptic-challenge matrix
- Global abort conditions
- Non-aborting degradation (add `## Angles we could not verify` section to README.md)

## Files This Skill Reads

- `C:\Users\noelh\.claude\skills\_research-shared\agent-prompts.md`
- `C:\Users\noelh\.claude\skills\_research-shared\verification-rubric.md`

Read with the Read tool at workflow start.

## Common Mistakes

- **Running in the vault directory.** Refuse and redirect to `/deep-research`. Don't produce vault-shaped pages via this skill.
- **Modifying repo code.** This skill only writes under `./research/`. Never edit source files — if research surfaces a fix, mention it in the README and stop there.
- **Creating a PR against `main` without a branch.** Always cut a `research/...` branch first.
- **Skipping the PR step when cwd is a git repo but the remote is missing.** Commit and push to local branch anyway; tell the user the PR step needs a remote.
- **Silently dropping failed angles.** Always surface them in `## Angles we could not verify`.
- **Assuming the researcher already wrote good URLs.** The Verifier is non-negotiable, even in shallow mode.
