# /ship-feature — Design

Date: 2026-04-21
Author: noel (with Claude)
Status: Draft, awaiting user review

## Purpose

A personal slash command that takes a one-line feature idea and drives it end-to-end — spec, plan, implementation, PR, CI, merge, cleanup — autonomously, with exactly one mandatory user checkpoint (spec approval) and automated per-task review gates.

## Non-goals

- Issue creation (the `prd-to-issues` skill already handles this upstream)
- Release notes / CHANGELOG updates
- Deployment or rollback
- Working on repos other than the one whose cwd the command is invoked from

## Invocation

```
/ship-feature <one-line feature description>
```

Argument is the idea as plain text. Everything else is derived or inferred.

## Checkpoint model

| Gate | Type | When |
|---|---|---|
| Spec approval | Hard block (user) | End of Phase 1 |
| Per-task spec + quality review | Hard block if fails | End of each task inside a concern-executor |
| Fabrication check on PR body | Hard block if fails ≥ 2× | Phase 5 |
| CI recovery exhaustion | Hard pause, report | After 3 failed fix attempts in Phase 6 |
| Worktree cleanup safety assertions | Hard block if fails | Phase 8 |

No checkpoint between spec-approved and merge — that is deliberate; auto mode.

## Settled decisions (answered by user 2026-04-21)

1. **Branching model: C** — N independent PRs, one per concern. Each concern ships in its own branch, its own worktree, its own PR, its own CI cycle.
2. **CI recovery bail threshold: 3 auto-fix attempts** before pausing to the user.
3. **Parallelism cap: `min(concerns, 3)`** concern-executor subagents running simultaneously, respecting dependency DAG.

## Phase-by-phase

### Phase 1 — Spec

Phase 1 runs in the orchestrator's own session rather than in a subagent, because the user wants the brainstormer to ask clarifying questions and subagents cannot prompt the user interactively.

1. Orchestrator invokes `superpowers:brainstorming` directly with the one-liner as seed material.
2. The skill runs its normal flow: explore repo, ask the user clarifying questions one-at-a-time, propose 2–3 approaches, present the design section by section, and write the spec to `<repo>/docs/specs/YYYY-MM-DD-<feature-slug>-spec.md`.
3. The brainstorming skill's built-in User Review Gate at the end of its flow **is** the Phase 1 approval checkpoint — no separate `AskUserQuestion`.
4. On revision requests: the brainstorming skill's own revise loop handles them in place. The orchestrator does not exit Phase 1 until brainstorming hands back `approved`.
5. On user abort during brainstorming: stop cleanly, leave any partial spec in place for later reuse.

Context-cost note: Phase 1 runs in-session, so tokens consumed here stay in the orchestrator's context through the rest of the run. Acceptable trade for interactive Q&A. If Phase 1 ever produces a spec longer than ~5k tokens, the orchestrator should read it back from disk rather than keeping it in scrollback before Phase 2.

### Phase 2 — Plan

1. Invoke `andrej-karpathy-skills:karpathy-guidelines` to set behavioral baseline (per `~/.claude/rules/common/development-workflow.md`).
2. Dispatch `planner` subagent with the approved spec path.
3. Subagent uses `superpowers:writing-plans` and returns a plan file at `<repo>/docs/plans/YYYY-MM-DD-<feature-slug>-plan.md` plus a parseable `concerns.json` emitted to the same directory:

    ```jsonc
    {
      "feature_slug": "add-search",
      "concerns": [
        {
          "slug": "index-pipeline",
          "branch": "feat/add-search-index-pipeline",
          "deps": [],
          "tasks": [
            {"id": "t1", "desc": "...", "tests_first": true, "acceptance": "..."}
          ],
          "pr_title": "feat(search): index pipeline"
        },
        {
          "slug": "query-api",
          "branch": "feat/add-search-query-api",
          "deps": ["index-pipeline"],
          "tasks": [...]
        }
      ]
    }
    ```

4. No user checkpoint (auto mode). Plan and `concerns.json` are written to the filesystem only — never committed. Subagents in later phases read them by absolute path, so the worktrees do not need their own copies.

### Phase 3 — Worktree per concern

1. Resolve `DEFAULT_BRANCH` once: `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`. Store for reuse in all later phases.
2. Run `git fetch origin` in the main repo.
3. For each concern, via `superpowers:using-git-worktrees`:
   - Create worktree at `<repo-parent>/<repo-name>-<concern.slug>/`
   - Branch `<concern.branch>` from `origin/<DEFAULT_BRANCH>`
4. **Worktree-cwd-guard assertions** per worktree, executed before any write:
   - `git rev-parse --abbrev-ref HEAD` ≠ `DEFAULT_BRANCH` and ∉ {`main`, `master`}
   - Worktree absolute path ≠ main-repo path
   - Worktree absolute path ≠ `$HOME`
   - Worktree absolute path appears in `git worktree list --porcelain`
5. On any assertion failure: abort the entire command and report.

### Phase 4 — Execute concerns (parallel, cap 3, DAG-ordered)

1. Scheduler reads `concerns.json`, topologically sorts, dispatches up to 3 `concern-executor` subagents at once. A concern is eligible when all its `deps` have reached "merged" status.
2. Each `concern-executor`:
   1. `cd` to its worktree (assertions re-checked)
   2. For each task in order:
      - `superpowers:test-driven-development` — write failing test, commit
      - Implement until green
      - `superpowers:verification-before-completion` — run full test suite + typecheck + lint, capture output
      - `superpowers:requesting-code-review` — dispatch two reviewers in parallel:
        - **spec-conformance reviewer**: reads the spec doc + the task's acceptance criteria + the diff, answers "does this satisfy the spec?"
        - **quality reviewer**: the `code-reviewer` agent
      - If either reviewer returns CRITICAL or HIGH, fix and re-review (max 2 iterations per task)
      - Commit the task with conventional-commits type
   3. Returns: `{status: "ready_for_pr" | "failed", branch, head_sha, commits, test_output, unresolved_review_findings}`

### Phase 5 — PR per concern

1. `superpowers:finishing-a-development-branch` drafts PR body from the concern's commits + acceptance criteria.
2. **Fabrication check** — dispatch `verify-pr-body` subagent with `{pr_body_draft, worktree_path, diff}`:
   - For every file path mentioned: `test -e` inside the worktree
   - For every function/symbol mentioned: `Grep` the worktree for its definition
   - For every flag or config key mentioned: `Grep` the worktree for it
   - For every URL mentioned: `WebFetch` it and require a 2xx
   - For every quoted log line or CLI output mentioned: verify it appears verbatim somewhere reachable
   - Returns: `{ok: bool, problems: [...]}`
3. On fail: the orchestrator prompts the drafter to rewrite without the offending content; max 2 attempts; still failing → pause for user.
4. On pass:
   - **Never-push-to-main guard**: assert `<concern.branch>` ∉ {`main`, `master`, `DEFAULT_BRANCH`}
   - `git push -u origin <concern.branch>`
   - `gh pr create --base <DEFAULT_BRANCH> --head <concern.branch> --title "<concern.pr_title>" --body <verified-body>`

### Phase 6 — CI recovery loop (per PR, max 3 iterations)

```
attempts = 0
while attempts < 3:
    result = gh pr checks <pr> --watch --fail-fast
    if result.all_green:
        break
    attempts += 1
    dispatch ci-fixer subagent with {pr_number, failure_logs, branch, worktree_path}
        subagent: diagnose, fix, run tests locally, commit, push
if attempts == 3 and not green:
    pause; report `gh pr view <pr> --json state,statusCheckRollup` + last fixer diff
```

### Phase 7 — Merge

- `gh pr merge <pr> --squash --auto --delete-branch`
- Concerns downstream of this one via `deps` become eligible; scheduler dispatches them if capacity allows.

### Phase 8 — Cleanup

1. For each worktree path `W` created in Phase 3:
   - **Worktree-cwd-guard** (second pass):
     - `pwd` ∉ `W`  (we are not standing inside it)
     - `W` ≠ main repo path
     - `W` ≠ `$HOME`
     - `W` ∈ `git worktree list --porcelain`
   - On pass: `git worktree remove <W>` (no `--force`)
   - On fail: skip + warn
2. Invoke `git-cleanup` skill against the main repo to prune merged local branches and remote-tracking refs.

## Skill & tool composition

Sequential backbone (orchestrator drives these in order):

| Phase | Primary skill / tool | New behavior the orchestrator adds |
|---|---|---|
| 1 | `superpowers:brainstorming` (in-session, not subagent) | none — use the skill's native Q&A + User Review Gate as the Phase 1 checkpoint |
| 2 | `andrej-karpathy-skills:karpathy-guidelines` → `superpowers:writing-plans` (inside planner subagent) | concerns-decomposition + `concerns.json` emission |
| 3 | `superpowers:using-git-worktrees` | worktree-cwd-guard assertions |
| 4 | `superpowers:executing-plans` + `subagent-driven-development` + `dispatching-parallel-agents` + `test-driven-development` + `verification-before-completion` + `requesting-code-review` | DAG scheduler with parallelism cap of 3; per-task spec-conformance reviewer |
| 5 | `superpowers:finishing-a-development-branch` + `gh` CLI | fabrication-check subagent (new); never-push-to-main guard |
| 6 | `gh` CLI | bounded CI recovery loop with fixer subagent |
| 7 | `gh` CLI | auto-merge via `--auto --delete-branch` |
| 8 | local `git-cleanup` skill | worktree-cwd-guard (second pass) |

Subagents dispatched by the orchestrator (each runs in an isolated context, returns a single structured message):

| Subagent | When | Input | Output |
|---|---|---|---|
| `planner` | Phase 2 | `{spec_path}` | `{plan_path, concerns_json_path}` |
| `concern-executor` | Phase 4 | `{concern, worktree_path, spec_path}` | `{status, branch, head_sha, test_output}` |
| spec-conformance reviewer | Inside concern-executor, per task | `{task, spec_path, diff}` | `{severity, findings}` |
| `verify-pr-body` | Phase 5 | `{pr_body_draft, worktree_path, diff}` | `{ok, problems}` |
| `ci-fixer` | Phase 6 (up to 3×) | `{pr_number, failure_logs, branch, worktree_path}` | `{diagnosis, commit_sha}` |

## Friction-point mitigations (maps to user's stated requirements)

| Requirement | Mitigation |
|---|---|
| Never push to main | Phase 5 branch-name assertion before `git push`; Phase 3 checks HEAD is not the default branch; `gh pr create` always passes `--base <DEFAULT_BRANCH> --head <concern.branch>` explicitly; `/ship-feature` refuses to run if cwd is not a clean repo on the default branch at invocation time |
| Separate branches per concern | Option C branching model: Phase 2 emits `concerns.json`; Phase 3 creates one worktree + branch per concern; Phase 5 opens one PR per concern |
| Verify worktree cwd before cleanup | Four-part assertion in Phase 8: pwd not inside target, target ≠ main repo, target ≠ `$HOME`, target present in `git worktree list` |
| Catch fabricated URLs/facts | `verify-pr-body` subagent in Phase 5: filesystem check for paths, grep check for symbols/flags, WebFetch for URLs, verbatim check for quoted output |

## Failure modes

| Failure | Behavior |
|---|---|
| Spec too large / multi-system | brainstorming skill's own decomposition guidance surfaces the problem during Phase 1 Q&A; orchestrator reports the suggested sub-projects and stops |
| Planner emits a concern with circular deps | Orchestrator rejects the plan and re-dispatches planner with the cycle error |
| Concern-executor fails a task after 2 review rounds | Concern marked failed; dependent concerns aborted; merged concerns preserved; user informed |
| Fabrication check fails 2× | Pause; show user the problems and the draft body |
| CI fails 3× | Pause; `gh pr view <pr>` output + last fixer diff reported |
| Worktree-cwd-guard fails in Phase 3 | Abort the entire command immediately |
| Worktree-cwd-guard fails in Phase 8 | Skip that worktree, warn, continue; leave the orphan for manual cleanup |
| User interrupts mid-run | All committed/merged state is preserved; command reports which concerns merged, which are in flight, which are blocked |

## File layout

- `~/.claude/skills/ship-feature/SKILL.md` — the orchestrator prompt (takes `$ARGUMENTS` = one-line feature description)
- `~/.claude/skills/ship-feature/design.md` — this document
- `<repo>/docs/specs/YYYY-MM-DD-<slug>-spec.md` — per-invocation spec (lives in the target repo)
- `<repo>/docs/plans/YYYY-MM-DD-<slug>-plan.md` — per-invocation plan (ditto)
- `<repo>/docs/plans/YYYY-MM-DD-<slug>-concerns.json` — machine-readable plan scheduler input

All subagents are prompt-only and inlined into `SKILL.md`. If one grows past ~40 lines, move it into a sibling file (`~/.claude/skills/ship-feature/<subagent>.md`) so the skill stays self-contained.

## Preconditions at invocation

`/ship-feature` refuses to run unless:

- cwd is inside a git repo
- working tree is clean (`git status --porcelain` empty)
- `DEFAULT_BRANCH` resolves via `gh repo view --json defaultBranchRef`
- HEAD is on `DEFAULT_BRANCH` and up to date with `origin/<DEFAULT_BRANCH>`
- `gh` CLI is authenticated (`gh auth status`)

Each precondition violation is reported with the exact fix command.

## Open questions

None currently. All three settled by user on 2026-04-21.
