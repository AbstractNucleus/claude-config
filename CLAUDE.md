# ~/.claude/ — synced Claude Code setup

This repo is the portable part of `~/.claude/`. Clone it on a new machine, and your skills, rules, agents, saved plans, and plugin list come with you. Auto-memory (per-project memory files Claude writes) is intentionally *not* synced — each machine builds its own as you work.

## What's tracked

| Path | What it is |
|------|------------|
| `settings.json` | Shared settings: permissions, hooks, model prefs, env vars |
| `skills/` | User-authored and installed skills |
| `rules/` | Global instruction rulesets (common + per-language) |
| `plans/` | Saved implementation plans |
| `docs/` | Design notes and reference docs |
| `plugins/blocklist.json` | Plugin blocklist |

Plugin configuration source-of-truth lives entirely in `settings.json`: `enabledPlugins` names which plugins to run, and `extraKnownMarketplaces` (plus the built-in `claude-plugins-official` marketplace) names where to fetch them from. Claude Code handles installation on first run.

## What's NOT tracked (and why)

- **`.credentials.json`, `config.json`** — contain OAuth tokens and API keys.
- **`settings.local.json`** — **machine-specific overrides** (see below).
- **`plugins/data/`, `plugins/cache/`, `plugins/marketplaces/`** — installed plugin content and cloned marketplace repos; re-fetched automatically.
- **`plugins/installed_plugins.json`, `plugins/known_marketplaces.json`** — both are rewritten at runtime with per-machine `installPath` / `installLocation` paths and timestamps, so tracking them causes churn. Source-of-truth for enablement lives in `settings.json → enabledPlugins`; for marketplace sources in `settings.json → extraKnownMarketplaces`.
- **`projects/`** — all per-project state (conversation history, todos, shell-snapshots, and auto-memory). Claude Code manages this directory; none of it benefits from syncing across machines.
- **Logs, caches, telemetry** — `bash-commands.log`, `cost-tracker.log`, `history.jsonl`, `cache/`, `paste-cache/`, `ide/`, `sessions/`, `telemetry/`, `usage-data/`, etc.
- **Stale backups** — `settings.json.bak`, `*.backup.*/`, `backups/`.

## Machine-specific overrides

**Rule:** `settings.json` is shared across machines. Put anything that differs per-machine into `settings.local.json` (gitignored).

Examples of what belongs in `settings.local.json`:
- Windows-only permissions (e.g. `powershell.exe` commands)
- macOS- or Linux-only shell paths
- Local-only env vars (e.g. an API endpoint pointing at a dev server)
- Per-machine model overrides

Claude Code merges `settings.local.json` on top of `settings.json` at load time, so local entries win. Never commit `settings.local.json`.

## Setting up a new machine

```bash
# 1. Clone into ~/.claude/ (must be empty or non-existent first)
git clone git@github.com:<you>/claude-config.git ~/.claude

# 2. Log in (recreates .credentials.json)
claude                   # prompts for auth on first run

# 3. Install plugins
#    Claude Code will install each plugin listed in
#    settings.json → enabledPlugins, fetching from marketplaces
#    defined in settings.json → extraKnownMarketplaces (plus the
#    built-in claude-plugins-official marketplace).
#    If any plugin is missing, install it manually:
#    /plugin install <name>

# 4. Create your machine-specific overrides
#    If you have OS-specific permissions or env vars, put them in:
#    ~/.claude/settings.local.json
```

## Updating the synced setup

Work in `~/.claude/` as normal — skills, rules, plans, and settings get updated in place. When you want to sync:

```bash
cd ~/.claude
git add -A
git status                # sanity-check nothing sensitive slipped in
git commit -m "update: <what changed>"
git push
```

On your other machines:

```bash
cd ~/.claude
git pull
```

## Safety

- **Before every commit**, scan `git status` for anything that looks like a token, key, or credential. The `.gitignore` covers known locations, but new Claude Code versions may add new files.
- The repo is **private**. Keep it that way. Saved plans and rulesets often contain project-specific context.
- If you ever commit a secret by accident: rotate it immediately, then rewrite history (`git filter-repo` or BFG) before the next push.
