---
name: query
description: Use when the user wants to ask a question against their Obsidian vault from any project — invoked via "/query <question>", "ask the vault", "look this up in my notes", or "what does my vault say about X". Spawns a sub-agent that loads the vault's own CLAUDE.md and runs the vault's `query` flow, so the main session never ingests vault context.
---

# Query

## Overview

Ask a question against the user's Obsidian vault at `C:\Users\noelh\Documents\vault` from **any** project. This skill exists to keep the main session's context window clean: it dispatches a sub-agent that loads the vault's `CLAUDE.md`, reads `index.md`, searches `pages/` (and `sources/` if needed), and returns only the synthesized answer plus citations.

The vault has its own `query` command defined in its `CLAUDE.md` — this skill is the *remote entry point* for that command from outside the vault directory.

## When to Use

- User types `/query <question>`
- User says "ask my vault", "what do my notes say about X", "look this up in the vault", "search my wiki for Y"
- User wants a fact, citation, or synthesis grounded in their own notes
- Out-of-scope: capturing new content (use `/note`), editing pages, running `ingest`/`promote`/`lint` (those happen inside the vault)

## Why a Sub-Agent

Token optimization. The vault's `CLAUDE.md`, `index.md`, and any page contents the agent reads can be large. If the main session loads them, every subsequent turn pays that token cost. By delegating to a sub-agent:

- The sub-agent's full context (vault rules, index, page reads) stays in *its* window.
- Only the final answer (a few hundred tokens with citations) returns to the main session.
- The main session's working context for the user's actual project is preserved.

## Workflow

1. Capture the user's question verbatim from `/query <...>` or surrounding chat.
2. If the question is empty or genuinely ambiguous, ask **one** clarifying question. Otherwise proceed.
3. Dispatch a sub-agent (Agent tool, `subagent_type: general-purpose`) with the prompt template below.
4. Relay the sub-agent's answer to the user. Preserve its `[[wikilinks]]` and `sources/...` citations as markdown links where useful, but do not add commentary the agent didn't produce.
5. If the sub-agent reports "not in vault," tell the user plainly — do not guess from outside knowledge.

## Sub-Agent Prompt Template

Pass this to the Agent tool. Fill in `{{question}}` with the user's actual question.

```
You are running the vault's `query` command on behalf of a main session that is NOT in the vault directory. Your job: answer the question using only the user's Obsidian vault, then return a concise answer with citations.

Working directory: C:\Users\noelh\Documents\vault

Steps:
1. Read C:\Users\noelh\Documents\vault\CLAUDE.md to load the vault's rules and conventions.
2. Read C:\Users\noelh\Documents\vault\index.md FIRST to orient on what pages exist.
3. Search pages/ (and sources/ if the answer needs raw material) for content relevant to the question.
4. Synthesize an answer grounded ONLY in what the vault contains. Do not import outside knowledge.
5. If the vault has no relevant content, say so explicitly — do not invent or extrapolate.

Question:
{{question}}

Return format:
- **Answer:** match the length to the question. A factual lookup gets 1–3 sentences. A "summarize everything about X" or "what do I know about Y" question gets the full picture — every relevant page surfaced, organized with subheadings if needed. Don't pad short answers; don't truncate broad ones.
- **Citations:** bullet list of [[page-name]] and/or sources/<filename> references you actually drew from. No citation = no claim. For breadth queries, cite every page touched.
- **Gaps (optional):** one line if the vault partially answers but is missing something obvious.
- **Suggested filing (optional):** if the synthesis is non-trivial and reusable, one line proposing a new page or an update to an existing page. Do NOT file it yourself — the main session will relay the suggestion to the user.

Calibrating length:
- "What's my take on X?" → tight answer
- "Summarize everything I have on X" / "all info about X" / "everything I've captured about X" → exhaustive — pull from every relevant page
- "List all my notes about X" → enumerate them all with one-line context each
- When in doubt, prefer completeness over brevity for breadth-shaped questions

Hard rules:
- Do not edit any vault file. Read-only.
- Do not run `ingest`, `promote`, or `lint`.
- Do not append to log.md.
- If you read a page, cite it. If you didn't read it, don't cite it.
```

## Returning the Answer

When the sub-agent returns:

- Show the **Answer** block as the primary response.
- Show the **Citations** as a short list so the user can click through (use markdown link form `[[page-name]]` or `vault/sources/file.md` paths).
- If a **Suggested filing** appeared, ask the user once: "Want me to file this as a new page / fold into `[[existing-page]]`?" If yes, the *user* re-enters the vault directory or you spawn a follow-up sub-agent to do the write — do not auto-file from here.

## Examples

### Simple lookup
User: `/query what's my take on linear attention?`

→ Spawn sub-agent with the question. Sub-agent reads `CLAUDE.md`, `index.md`, finds `[[linear-attention]]` and any inbox notes, returns:

> **Answer:** You flagged linear attention as the next bottleneck after KV-cache tricks (note from 2026-04-13). The vault page `[[linear-attention]]` summarizes the Performer and Linformer approaches and notes a tradeoff between recall quality and memory.
> **Citations:** [[linear-attention]], inbox/2026-04-13-1432-linear-attention-next-bottleneck.md

### Question with no vault coverage
User: `/query what does my vault say about Mamba state-space models?`

→ Sub-agent returns: "No pages in the vault mention Mamba or state-space models. `[[linear-attention]]` is the closest adjacent topic."

→ Relay verbatim. Do not fill the gap from training data.

### Synthesis-worthy answer
User: `/query summarize everything I've captured about Karpathy`

→ Sub-agent returns answer + citations + a **Suggested filing** line proposing a `[[karpathy]]` page consolidating scattered references.

→ Show the answer. Ask the user once whether to file.

## Quick Reference

| Scenario | Action |
|----------|--------|
| `/query <question>` | Dispatch sub-agent with question verbatim |
| Empty `/query` | Ask once: "What do you want to look up?" |
| Vault directory missing | Tell user, don't auto-create |
| Sub-agent says "not in vault" | Relay plainly, don't guess |
| Answer is reusable | Surface the suggested-filing line, ask once |

## What NOT to do

- **Don't read vault files in the main session.** That defeats the purpose. Always delegate.
- **Don't auto-file** the answer as a new page. The vault's `query` flow asks the user first; this skill preserves that.
- **Don't merge with outside knowledge.** If the vault doesn't say it, the answer doesn't say it.
- **Don't truncate citations.** They are the user's audit trail.
- **Don't run `ingest`/`promote`/`lint`** — those are explicit vault-side commands.
- **Don't ask multiple clarifying questions.** One, max, only if the question is genuinely empty or ambiguous.

## Vault Path

Hardcoded: `C:\Users\noelh\Documents\vault`

If the vault path ever moves, update this skill and the sub-agent prompt template above.
