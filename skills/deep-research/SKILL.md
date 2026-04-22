---
name: deep-research
description: Use when the user wants rigorous, multi-angle research written into their Obsidian vault — invoked via "/deep-research <topic>", "research this for my vault", "do a deep dive on X". Decomposes into 5–7 parallel angles, dispatches researcher subagents with strict URL-per-claim citations, verifies every URL by WebFetch, runs a skeptic to challenge weak claims, re-researches on demand, and synthesizes only verified content into a vault-template synthesis page. Writes raw research to sources/ and synthesis draft to inbox/, then opens a PR on the vault repo.
argument-hint: [shallow] <topic>
---

# Deep Research — Vault Mode

## Overview

End-to-end pipeline that produces a structured research synthesis in the user's Obsidian vault. Uses parallel subagents for decomposition, investigation, verification, adversarial review, and synthesis. Every claim in the final synthesis is backed by a URL that was WebFetched and verified.

Output shape matches the vault's existing research template ([[idle-game-design]]): wikilinks in the body, URLs live in `sources/` files referenced in front-matter.

## When to Use

- User types `/deep-research <topic>`
- User says "research this for my vault", "deep dive X for my notes", "add X to my wiki"
- User wants reusable knowledge filed into their Obsidian vault
- Out-of-scope: project-scoped research (use `/research-here` instead), quick factual lookups (use `/query`), capturing user's own thoughts (use `/note`)

## Argument Parsing

`argument-hint: [shallow] <topic>`

- `shallow <topic>` — skip the skeptic + re-research loop. Use for low-stakes topics or quick scoping passes.
- `<topic>` — default (deep). Include skeptic + re-research on `high`/`medium` challenges.

If the first positional arg is exactly `shallow`, treat the rest as the topic. Otherwise the whole arg string is the topic.

## Safety Posture

- If cwd is inside the vault directory (`C:\Users\noelh\Documents\vault`), warn once: "You're already in the vault — the PR step may conflict with your working state. Proceed?" then continue if confirmed.
- If cwd is clearly a project repo (contains `package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`, etc.), warn once: "You're in a project directory — did you mean `/research-here`?" then proceed to vault mode if confirmed.
- Never write to `pages/` directly. Only `sources/` + `inbox/` — the user runs vault-side `promote` when they're ready.
- Never run vault-side `ingest`, `promote`, or `lint` from this skill. Those are explicit vault commands.

## Workflow

### 1. Parse args

- Extract `depth ∈ {shallow, deep}` (default `deep`) and `topic` from the argument string.
- If `topic` is empty, ask once: "What topic do you want researched?"
- Compute `topic_slug` (kebab-case of topic, ≤ 60 chars).
- Compute `stamp = YYYY-MM-DD-HHMM`, `today = YYYY-MM-DD`.

### 2. Decompose

- Read `./agent-prompts.md` §Decomposer.
- Dispatch one Agent call with the Decomposer prompt, substituting `{{topic}}` and any context the user already provided.
- Parse the JSON. Validate: 5–7 angles, all have non-empty `boundaries`.
- Announce to the user: "Decomposed into N angles: <titles>." Proceed without confirmation — auto mode.

### 3. Research (parallel)

- Read `./agent-prompts.md` §Researcher.
- Dispatch ONE Agent call per angle, all in a single message (parallel).
  - `subagent_type: general-purpose`
  - `description: Research angle N — <title>`
  - Pass each researcher the Researcher prompt with `{{topic}}`, `{{angle.*}}` substituted.
- Collect all outputs. Keep them addressable by angle id.

### 4. Verify

- Read `./agent-prompts.md` §Verifier and `./verification-rubric.md`.
- Dispatch one Agent call with the Verifier prompt and the concatenated researcher outputs.
- Parse the verdict JSON.
- Check global abort conditions from the rubric (`>40% DEAD/HALLUCINATED` etc.). If triggered, stop and report to the user — do not synthesize.

### 5. Skeptic (deep only)

- If `depth == shallow`, skip to step 7.
- Read `./agent-prompts.md` §Skeptic.
- Dispatch one Agent call with researcher outputs + verifier JSON.
- Parse challenges JSON.

### 6. Re-research (deep only)

- For each `high`-severity challenge, dispatch a Re-researcher (parallel).
- For `medium`-severity, dispatch in parallel with synthesis scaffolding (or sequentially — either is acceptable).
- For `low`-severity, skip unless topic is small enough that it's cheap to include.
- Collect re-research outcomes. Apply the rubric's skeptic-handling matrix: RESOLVED → keep strengthened, STILL-WEAK → Open questions, CONFIRMED-COUNTER → retract.

### 7. Vault index

- Read `./agent-prompts.md` §Vault-indexer.
- Dispatch the Vault-indexer Agent. It reads the vault's `CLAUDE.md` + `index.md` and returns a page map.
- This runs inside a subagent so the main session never loads vault context directly.

### 8. Synthesize

- Read `./agent-prompts.md` §Synthesizer — vault mode.
- Also fetch the template page `pages/idle-game-design.md` content via the Vault-indexer (or a follow-up subagent read) and pass it in as `template_page_content`.
- Filter researcher claims: include only those where at least one cited URL got an OK verdict from the Verifier AND the claim survived the Skeptic (in deep mode).
- Dispatch the Synthesizer Agent with the filtered content and vault page map.
- Receive the synthesized markdown.

### 9. Write files

Write to the vault at `C:\Users\noelh\Documents\vault`:

```
sources/{stamp}-deep-research-{topic_slug}-a1-{angle1_slug}.md
sources/{stamp}-deep-research-{topic_slug}-a2-{angle2_slug}.md
...
sources/{stamp}-deep-research-{topic_slug}-verification.md
inbox/{stamp}-research-{topic_slug}.md
```

- Each `angle-aN` file: the raw researcher output (keeps URLs).
- The `verification.md` file: verifier JSON + skeptic JSON + re-research outputs, as readable audit trail.
- The `inbox/` file: the synthesizer's output. Front-matter's `sources:` array lists every `angle-aN` filename AND the `verification.md` filename.

### 10. Git + PR

```bash
cd C:\Users\noelh\Documents\vault
git checkout -b research/{topic_slug}-{YYYY-MM-DD}
git add sources/{stamp}-deep-research-{topic_slug}-*.md
git add inbox/{stamp}-research-{topic_slug}.md
git commit -m "research: {topic} (deep-research)"
git push -u origin research/{topic_slug}-{YYYY-MM-DD}
gh pr create --title "research: {topic}" --body "<summary>"
```

PR body (HEREDOC):

```markdown
## Summary
- Topic: {topic}
- Depth: {depth}
- Angles: {N}
- Verifier: {ok}/{total} OK, {weak} WEAK, {dead} DEAD, {hallucinated} HALLUCINATED
- Skeptic (if deep): {overall_assessment}, {N} challenges, {N} resolved

## Files
- sources/ … (list)
- inbox/{stamp}-research-{topic_slug}.md (synthesis draft — run `promote` when ready)

## Open questions
- <from skeptic, if any>
```

### 11. Report

Tell the user:
- PR URL
- File paths written
- Verifier summary (N OK / N WEAK / N DEAD / N HALLUCINATED)
- Skeptic verdict if deep
- Any angles that failed verification (never silently dropped — surface them)

## Partial Failure Handling

Follow `./verification-rubric.md` exactly:
- Per-claim failure matrix for verifier verdicts
- Skeptic-challenge matrix for challenge severity
- Global abort conditions
- Non-aborting degradation (`## Angles we could not verify` section when 1–2 angles fail hard)

## Files This Skill Reads

- `C:\Users\noelh\.claude\skills\deep-research\agent-prompts.md` — all subagent prompts
- `C:\Users\noelh\.claude\skills\deep-research\verification-rubric.md` — rubric + failure matrix

Read them with the Read tool at workflow start. Do not cache across runs.

## Common Mistakes

- **Reading vault pages in the main session.** Defeats the point of the Vault-indexer subagent — loads the vault into context. Always delegate vault reads.
- **Copying the template's content.** The synthesizer matches the template's SHAPE (headings style, section order, front-matter), not its topic.
- **Dropping failed angles silently.** If an angle fails verification, surface it in `## Angles we could not verify`, don't just omit.
- **Writing to `pages/` directly.** Only `sources/` + `inbox/`. The vault's `promote` command turns inbox drafts into pages.
- **Skipping the PR when cwd is inside the vault.** If the user is already working in the vault, running this skill may conflict with their working tree. Warn first.
- **Running with `depth=deep` on topics the web can't support.** If the Verifier's global-abort conditions trigger, stop — don't try to synthesize from hallucinated claims.
