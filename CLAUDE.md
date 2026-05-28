# ~/.claude/ synced config repo

Git repo that syncs Claude Code config across machines. `settings.json`, `skills/`, `CLAUDE.md`, and `README.md` are tracked. Everything else (credentials, install state, plugins, conversation history, logs, caches) is gitignored. See `.gitignore` for the authoritative list.

**Config split:** `settings.json` holds shared config (permissions, hooks, model prefs). If a machine needs overrides, create `settings.local.json`. It's gitignored and merged on top at load time. Skills that need per-machine values (vault path, deploy hosts, design preferences) store them in their own folder via a gitignored config file with a committed `.example.md` template.

**Committing here:** before any commit, scan `git status` for secrets or unexpected new files. `.gitignore` may lag new Claude Code additions. If a secret is ever committed, rotate it before rewriting history.

# Coding guidelines

Always apply the `karpathy-guidelines` skill when writing, reviewing, or refactoring code in any project. Keep changes surgical, surface assumptions, avoid overcomplication, and define verifiable success criteria. Invoke the skill via the Skill tool at the start of any non-trivial coding task so its full guidance is in context.

# Self-improvement loop

After any correction, surfaced friction, or mistake in a session, ask: where should this lesson live so it doesn't repeat?

- Mechanical guarantee that must run every time → a hook in `.claude/settings.json`, not a memory.
- Cross-project always-on style → this file.
- Behavior specific to one project → that project's `CLAUDE.md`.
- File-type-scoped advisory rule → `~/.claude/rules/` with `paths:` frontmatter (small + few; re-injects per tool call).
- Reusable expertise across projects, on-demand → a skill in `~/.claude/skills/`.

For skills specifically: edit `SKILL.md` directly to fix the unclear step, cut dead prose, or sharpen the trigger. Keep skills lean; cut when in doubt. If a fix isn't obvious, surface the proposed change before editing.

Only self-edit skills under `~/.claude/skills/`. Do not edit Anthropic-shipped or plugin-namespaced skills (`anthropic-skills:*`, etc.). Flag improvement ideas instead.
