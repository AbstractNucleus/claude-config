# ~/.claude/ synced config repo

Git repo that syncs Claude Code config across machines. `settings.json`, `skills/`, `CLAUDE.md`, and `README.md` are tracked. Everything else (credentials, install state, plugins, conversation history, logs, caches) is gitignored. See `.gitignore` for the authoritative list.

**Config split:** `settings.json` holds shared config (permissions, hooks, model prefs). If a machine needs overrides, create `settings.local.json`. It's gitignored and merged on top at load time. Skills that need per-machine values (vault path, deploy hosts, design preferences) store them in their own folder via a gitignored config file with a committed `.example.md` template.

**Committing here:** before any commit, scan `git status` for secrets or unexpected new files. `.gitignore` may lag new Claude Code additions. If a secret is ever committed, rotate it before rewriting history.

# Coding guidelines

Always apply the `karpathy-guidelines` skill when writing, reviewing, or refactoring code in any project. Keep changes surgical, surface assumptions, avoid overcomplication, and define verifiable success criteria. Invoke the skill via the Skill tool at the start of any non-trivial coding task so its full guidance is in context.

# Self-improving skills

After invoking any skill from `~/.claude/skills/`, take a moment to ask: did anything block me, mislead me, or feel like bloat? If yes, edit the skill's `SKILL.md` directly to fix the unclear step, cut dead prose, or sharpen the trigger. Keep skills lean; cut when in doubt. If a fix isn't obvious, surface the proposed change to the user before editing.

Only self-edit skills under `~/.claude/skills/`. Do not edit Anthropic-shipped or plugin-namespaced skills (`anthropic-skills:*`, etc.). Flag improvement ideas to the user instead.
