---
name: feedback_nano
description: User prefers nano over cat/tee heredocs when editing files on their Linux machine
type: feedback
---

Use `nano` to edit/create files instead of multiline `cat` or `tee` heredoc commands.
**Why:** User finds it easier to paste content into nano than to deal with heredoc syntax in the terminal.
**How to apply:** When giving instructions to create or edit files on their Arch system, always use `sudo nano <path>` and show the content to paste, rather than `cat >` or `tee <<` commands.
