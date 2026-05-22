---
name: note
description: Use when the user wants to capture content into their Obsidian vault from any project, invoked via "/note", "note this", "save to vault", or by passing inline text, a file path, or a URL with capture intent. Routes to inbox (own thoughts) or sources (external material).
---

# Note

## Overview

Capture content into the user's Obsidian vault. This is the *capture* step only, it drops content into `inbox/` or `sources/` inside the vault. Processing into the curated wiki happens later via the vault's own `ingest` / `promote` commands (defined in the vault's `CLAUDE.md`).

This skill works from any project. It does not require the user's current working directory to be the vault.

## Setup

This skill needs to know where the vault is. The path is stored in `VAULT.md` inside this skill's folder (gitignored, per-machine).

At the start of every run:

1. Try to read `VAULT.md` from this skill's folder. Strip blank lines and lines starting with `#`. The first remaining line is the vault path. If it points to a directory that exists, use it as `{vault_path}` and proceed.
2. If `VAULT.md` is missing, empty (after stripping comments), or points to a non-existent directory: ask the user **once**, "Where's your Obsidian vault? Give me the absolute path." Write the answer to `VAULT.md` as a single line and continue. If the user says they don't have a vault, tell them this skill needs one and stop.
3. The path may use either platform separator. Use it verbatim, don't try to normalize across OSes.

The rest of this document refers to the vault as `{vault_path}`. Substitute the real path wherever you see it.

## When to Use

- User types `/note ...`
- User says "note this", "save this to my vault", "capture this", "drop this in my vault"
- User pastes text or a quote and asks to file it
- User points to a file (PDF, markdown, transcript, screenshot) or URL and asks to save it
- Out-of-scope: editing existing wiki pages, answering vault queries, running `ingest`/`promote`/`lint`, those happen inside the vault directory

## Routing

| Input | Destination | Why |
|-------|-------------|-----|
| Inline text from user | `inbox/` | The user wrote it, it's a fleeting thought |
| File path the user references | `sources/` | External material, treat as raw input |
| URL | `sources/` | External material |
| Ambiguous | Ask one question, then proceed |

**Override syntax:** user can prefix the note with `inbox:` or `sources:` to force routing.
- `/note sources: this is actually a quote from a book I'm reading` → `sources/`
- `/note inbox: D:\my-draft.md` → `inbox/`

## Filename Convention

**Inbox notes** (always `.md`):
```
YYYY-MM-DD-HHMM-<slug>.md
```
- Slug: 3–6 word kebab-case summary
- Example: `2026-04-13-1432-revisit-karpathy-bpe-video.md`

**Sources** (preserve original extension; `.md` for URL captures):
```
YYYY-MM-DD-<slug>.<ext>
```
- Slug: derived from title, headline, or original filename
- Example: `2026-04-13-karpathy-llm-wiki.md`, `2026-04-13-attention-is-all-you-need.pdf`

## Frontmatter

Inbox markdown notes get:
```yaml
---
captured: YYYY-MM-DD HH:MM
from: <project name or "chat">
ref: <path or URL the note is about, optional>
---
```

Sources markdown (URL captures or text clippings) get:
```yaml
---
captured: YYYY-MM-DD HH:MM
url: <original URL if any>
title: <article title if known>
---
```

Binary files (PDF, images) are copied as-is, no frontmatter.

### The `ref` field

`ref` is the **address of the subject** the note is about, a repo path, a file path, a URL, a directory. It's optional but valuable: when the vault later runs `ingest`/`promote` on the note, the wiki workflow reads `ref` and can crawl the referenced location for more context (READMEs, file structure, package manifests) to build a richer page.

**When to capture a `ref`:**
- The user mentions a path-shaped or URL-shaped token next to the subject name
- The note is *about* a project, repo, file, or website that exists somewhere

**Detection patterns** (be liberal, pull anything path- or URL-shaped from the subject line):
- `<name> at <path>`
- `<name> - <path>`
- `<name> (<path>)`
- `<name> <url>`

`ref` accepts a single path/URL string. If the user gives multiple, pick the most authoritative (repo > local clone > URL) and put the rest in the body.

## Workflow

1. Resolve `{vault_path}` per the Setup section above.
2. Parse intent: inline text vs file path vs URL. Check for `inbox:` / `sources:` override.
3. Check for multiple subjects (see **Splitting** below). If present, plan one file per subject.
4. Pick destination per the routing table.
5. Generate filename(s) per convention. Use the local date/time (not UTC). Same `HHMM` is fine across split files, slug differentiates.
6. Write/copy the file(s) under `{vault_path}/inbox/` or `{vault_path}/sources/`. For URLs, fetch with WebFetch and save the markdown content.
7. Confirm with the user, list every absolute path written.
8. **Stop.** Do not auto-ingest, do not edit `index.md` or `log.md`, do not touch `pages/`.

## Splitting multi-subject notes

When the user's input clearly covers **multiple distinct subjects of the same kind**, write one file per subject instead of one combined file. The vault's wiki layer wants one entity per page, splitting at capture time makes the later `promote` step trivial.

**Strong signals to split** (default: split, no need to ask):
- Plural collective prefix: `/note projects: ...`, `/note papers: ...`, `/note ideas on X and Y`
- The body has multiple top-level headings (`# Foo`, `# Bar`) each naming a distinct subject
- Bulleted/numbered list where each item is itself a multi-sentence subject
- "Three things:" / "two projects:" / explicit count followed by a list

**Don't split** when:
- The input is a single thought that happens to mention several names ("X reminded me of Y because of Z")
- It's a quote or excerpt, quotes stay whole
- It's a single file or URL, even if it covers multiple topics, the artifact is one file

**Ambiguous?** Default to one note. The user can re-run `/note` per subject if they wanted splits.

**Per-split filename:** each gets its own slug derived from that subject. Same date and time across the set is fine.

**Per-split content:** preserve the user's text *for that subject* verbatim. You may strip the prefix (`projects:`) and the heading that names the subject if it's redundant with the filename, but never rewrite the substance.

**Per-split `ref`:** if a path or URL appears alongside a subject, pull it into that split file's `ref:` frontmatter so a future `ingest` can crawl the location for richer context.

## Examples

### Inline thought
User: `/note I should read more on linear attention, feels like the next bottleneck after KV cache tricks`

→ Write `{vault_path}/inbox/2026-04-13-1432-linear-attention-next-bottleneck.md`:
```markdown
---
captured: 2026-04-13 14:32
from: chat
---

I should read more on linear attention, feels like the next bottleneck after KV cache tricks
```

### File capture
User: `/note D:\downloads\paper.pdf`

→ Copy to `{vault_path}/sources/2026-04-13-paper.pdf`. Confirm path.

### URL capture
User: `/note https://example.com/article-on-rope`

→ Fetch URL, save markdown to `{vault_path}/sources/2026-04-13-article-on-rope.md` with frontmatter containing the original URL.

### Override
User: `/note sources: "Talent hits a target no one else can hit; genius hits a target no one else can see." (Schopenhauer)`

→ Write to `{vault_path}/sources/2026-04-13-schopenhauer-genius-quote.md`.

### Multi-subject split with refs
User:
```
/note projects:
- proj-alpha at D:\code\proj-alpha, short description of project A
- proj-beta at D:\code\proj-beta, short description of project B
- proj-gamma at https://github.com/<user>/proj-gamma, short description of project C
```

→ Three files in `inbox/`, same timestamp, distinct slugs, each with its own `ref:`:

`2026-04-13-1505-proj-alpha.md`:
```markdown
---
captured: 2026-04-13 15:05
from: chat
ref: D:\code\proj-alpha
---

short description of project A
```

(Two more files in the same shape, one per remaining subject.)

The path was extracted from each subject line into `ref:` so the vault's later `promote` can crawl those locations for richer context. Confirm by listing all paths written.

## Quick Reference

| Scenario | Action |
|----------|--------|
| Plain text | inbox/ as `.md` |
| Local file path | sources/, copy as-is |
| URL | sources/ as `.md` after WebFetch |
| Prefix `inbox:` or `sources:` | Force destination |
| Plural prefix (`projects:`, `papers:`) or multiple top-level headings | Split → one file per subject |
| Path doesn't exist | Tell user, don't guess |
| `VAULT.md` missing/invalid | Ask user for path, save, continue |

## What NOT to do

- **Don't write to `pages/`**, that's the curated wiki, owned by the vault's own `ingest`/`promote` flow.
- **Don't run `ingest` or `promote`**, those are explicit user commands inside the vault.
- **Don't edit `index.md` or `log.md`**, same reason.
- **Don't ask 5 clarifying questions**, pick a sensible default per the routing table and tell the user what you did. One question max if routing is genuinely ambiguous.
- **Don't summarize or rewrite the user's text** when capturing to inbox, preserve it verbatim. The user wrote it, that's the artifact.
- **Don't capture without showing the path**, always confirm where you wrote.
