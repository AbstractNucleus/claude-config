# ~/.claude/ — synced config repo

Git repo that syncs Claude Code config across machines. `settings.json` and `skills/` are tracked; `.credentials.json`, `settings.local.json`, `plugins/`, `projects/`, logs, and caches are not — see `.gitignore` for the authoritative list. Human-facing setup and sync how-to lives in `README.md`.

**Config split:** `settings.json` holds shared config (permissions, hooks, `enabledPlugins`, `extraKnownMarketplaces`). `settings.local.json` holds per-machine overrides (OS-specific paths, local env vars, dev endpoints) and is gitignored. The local file is merged on top at load time, so local entries win — put anything machine-specific there, never in `settings.json`.

**Committing here:** never stage `.credentials.json`, `config.json`, or `settings.local.json`. Before any commit, scan `git status` for tokens or unexpected new files; `.gitignore` may lag new Claude Code additions. If a secret is ever committed, rotate it before rewriting history.
