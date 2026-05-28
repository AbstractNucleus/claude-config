# Claude Code skills

A small set of [Claude Code](https://claude.com/claude-code) skills I use day-to-day. Each one lives in its own folder under [`skills/`](skills/) and is self-contained. You can copy any single skill into your own `~/.claude/skills/` without taking the rest.

## What's in here

### Building and shipping

- [`diagnose`](skills/diagnose/) runs a disciplined bug-diagnosis loop: reproduce, minimise, hypothesise, instrument, fix, regression-test.
- [`improve`](skills/improve/) picks one worthwhile change for the current project, proposes it, then either grills you on it or implements it.
- [`improve-codebase-architecture`](skills/improve-codebase-architecture/) finds deepening opportunities and walks the design tree with you.
- [`karpathy-guidelines`](skills/karpathy-guidelines/) is a short set of coding behavioral rules to reduce common LLM mistakes, adapted from Andrej Karpathy's principles.
- [`frontend-design`](skills/frontend-design/) biases UI work toward distinctive, polished output. Reads your stack preferences from a local `PREFERENCES.md`.
- [`deploy`](skills/deploy/) ships the current project over SSH to hosts you list in `HOSTS.md`.

### Planning

- [`grill-me`](skills/grill-me/) interviews you about a plan or design until every branch of the decision tree is resolved.
- [`to-prd`](skills/to-prd/) turns the current conversation into a PRD on your issue tracker.
- [`to-issues`](skills/to-issues/) breaks a plan into independently-grabbable tickets using tracer-bullet slices.

### Research and notes

- [`research-here`](skills/research-here/) does multi-angle research with URL-per-claim citations and writes the synthesis into `./research/` in the current repo.
- [`note`](skills/note/) captures content into an Obsidian vault. The vault path lives in a gitignored `VAULT.md` per machine.
- [`query`](skills/query/) asks a question against the same vault via a sub-agent, so the main session never loads vault context.

### Workflow

- [`git-cleanup`](skills/git-cleanup/) prunes merged local branches.
- [`caveman`](skills/caveman/) switches into an ultra-compressed communication mode (≈75% fewer tokens).

## Using the whole set

```bash
git clone https://github.com/AbstractNucleus/claude-config.git ~/.claude
claude  # authenticates on first run
```

Skills that need a path or preference (`deploy`, `note`, `query`, `frontend-design`) will ask you for it the first time they run, and save your answer to a gitignored file in that skill's folder.

## Using a single skill

If you only want one:

```bash
cp -r path/to/this/repo/skills/diagnose ~/.claude/skills/
```

Restart Claude Code. The skill self-registers via its `SKILL.md` frontmatter.

## Repo layout

This repo doubles as a synced `~/.claude/` — cloning it there carries skills and config across machines. Tracked: `skills/`, `CLAUDE.md`, `README.md`. Everything else is gitignored: credentials, install state, plugins, conversation history, logs, caches, and `settings.json`. [`.gitignore`](.gitignore) is the authoritative list.

`settings.json` is **per-machine, not synced** — Claude Code rewrites UI/runtime toggles (editor mode, remote-control, notifications) back into it on every change, which caused constant cross-machine churn. So each machine keeps its own, and shared config (permissions, hooks, model) is set up per machine.

Before committing, scan `git status` for secrets or unexpected new files (`.gitignore` can lag new Claude Code additions). If a secret ever lands in a commit, rotate it before rewriting history.

## Design notes

- **Standalone.** No skill calls another skill by name. You can delete or copy any folder without breaking the rest.
- **No machine-specific values committed.** Per-machine config (vault paths, deploy hosts, design preferences) lives in gitignored files alongside each skill, with a committed `.example.md` template.
- **No plugin dependencies.** Everything you need is in this repo. `settings.json` doesn't enable any plugins.

## Credits

[`karpathy-guidelines`](skills/karpathy-guidelines/) is adapted from [forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills) (MIT).

## License

MIT. See [LICENSE](LICENSE).
