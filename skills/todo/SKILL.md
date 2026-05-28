---
name: todo
description: Append a todo item to `todo.md` in the current project root. Use when the user invokes `/todo <text>`, says "add a todo", "write down this todo", "track this", or similar. Creates `todo.md` if missing. Capture only — does not check off, reorder, or summarize existing todos.
---

# Todo

## Overview

Append a single todo item to `todo.md` at the current working directory (the project root for the session). Create the file if it doesn't exist. Confirm the path written.

## When to Use

- User types `/todo <text>` — the text after `/todo` is the todo body
- User says "add a todo: …", "write this down as a todo", "track this", "remind me to …"
- Out-of-scope: marking items done, editing/reordering existing items, summarizing the list, syncing to an external tracker. For those, edit `todo.md` directly or use `to-issues`.

## Workflow

1. Take the user's argument as the todo body. Strip surrounding whitespace and a leading `-` or `*` if the user pre-typed a bullet. You may lightly rephrase for clarity or concision (tighten wording, fix obvious typos, drop filler) — but preserve the user's intent, any specific identifiers (file paths, function names, error strings, ticket IDs, URLs), and don't expand scope or add detail the user didn't give. If the original is already tight, leave it alone.
2. Resolve the target path as `todo.md` in the current working directory.
3. If `todo.md` does not exist, create it with a single `# Todo` heading followed by a blank line, then the new item.
4. If `todo.md` exists, append the new item on its own line at the end of the file. Ensure the file ends with exactly one trailing newline after the append.
5. Format each item as a GitHub-style task list entry:
   ```
   - [ ] <body>
   ```
6. Confirm with one short line: the absolute path written and the item appended.

## Multiple todos in one invocation

If the user's input contains multiple distinct todos (multi-line input, or a numbered/bulleted list where each item is its own todo), append one `- [ ] <body>` line per todo, in the order given. Otherwise treat the whole input as a single todo.

Ambiguous? Default to one todo. The user can re-run.

## Examples

### First todo in a fresh project
User: `/todo fix navbar overflow on mobile`

`todo.md` does not exist. Write:
```markdown
# Todo

- [ ] fix navbar overflow on mobile
```

Reply: `Appended to <abs path>\todo.md: fix navbar overflow on mobile`

### Subsequent todo
User: `/todo write a test for the auth middleware`

`todo.md` exists. Append `- [ ] write a test for the auth middleware` at the end.

### Multi-todo
User:
```
/todo
- bump node to 22
- regenerate lockfile
- run CI
```

Append three lines, one per todo, in order.

## What NOT to do

- **Don't change the meaning, scope, or specifics** of the user's text. Light rephrasing for clarity is fine; rewriting into a different task is not.
- **Don't check off or remove existing items** — capture only.
- **Don't add timestamps, priorities, or tags** unless the user included them in the body.
- **Don't create `todo.md` anywhere other than the current working directory** — no walking up to a git root, no other locations.
- **Don't ask clarifying questions** for a non-empty body. If the body is empty, ask once for the todo text.
