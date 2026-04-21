---
description: Ship a feature end-to-end — spec, plan, implement, PR, merge, cleanup — with one user checkpoint at spec approval
argument-hint: <one-line feature description>
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, Agent, Skill, AskUserQuestion, WebFetch, TaskCreate, TaskUpdate
---

You are running the `/ship-feature` orchestrator. The user's one-line feature description is:

$ARGUMENTS

Follow the design at `~/.claude/docs/ship-feature-design.md`. Execute phases in order. Create a TaskCreate todo for every phase so the user can see progress.

## Phase 0: Preconditions

Run these checks IN ORDER. Bail immediately on any failure, printing the exact fix command.

```bash
# 0.1 cwd is inside a git repo
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || {
  echo "FAIL: /ship-feature must be run inside a git repo."
  echo "FIX: cd into a repo and retry."
  exit 1
}

# 0.2 working tree is clean
if [ -n "$(git status --porcelain)" ]; then
  echo "FAIL: working tree is not clean."
  echo "FIX: commit or stash your changes, then retry."
  exit 1
fi

# 0.3 gh is authenticated
gh auth status >/dev/null 2>&1 || {
  echo "FAIL: gh CLI is not authenticated."
  echo "FIX: gh auth login"
  exit 1
}

# 0.4 resolve default branch (export for later phases)
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name)
[ -n "$DEFAULT_BRANCH" ] || {
  echo "FAIL: could not resolve default branch from gh."
  echo "FIX: ensure this repo has a remote on GitHub."
  exit 1
}

# 0.5 HEAD is on the default branch and up to date with origin
CURRENT=$(git rev-parse --abbrev-ref HEAD)
if [ "$CURRENT" != "$DEFAULT_BRANCH" ]; then
  echo "FAIL: HEAD is on '$CURRENT', not '$DEFAULT_BRANCH'."
  echo "FIX: git checkout $DEFAULT_BRANCH"
  exit 1
fi
git fetch origin "$DEFAULT_BRANCH" --quiet || {
  echo "FAIL: could not fetch origin/$DEFAULT_BRANCH."
  echo "FIX: check network connectivity and ensure origin is reachable."
  exit 1
}
LOCAL=$(git rev-parse HEAD)
REMOTE=$(git rev-parse "origin/$DEFAULT_BRANCH")
if [ "$LOCAL" != "$REMOTE" ]; then
  echo "FAIL: local $DEFAULT_BRANCH is not in sync with origin/$DEFAULT_BRANCH."
  echo "FIX: git pull --ff-only"
  exit 1
fi

echo "Preconditions OK. DEFAULT_BRANCH=$DEFAULT_BRANCH"
```

Record `DEFAULT_BRANCH` in orchestrator state — every later phase uses it.

## Phase 1: Spec

Phase 1 runs in THIS session (not a subagent) so brainstorming can ask the user clarifying questions. The brainstorming skill's own User Review Gate is the Phase 1 approval checkpoint — do not add a second `AskUserQuestion`.

1. Invoke the `superpowers:brainstorming` skill with the user's one-liner ($ARGUMENTS) as seed material. Let the skill run its full native flow (explore repo → ask clarifying questions one-at-a-time → propose 2–3 approaches → present design section-by-section → write spec → User Review Gate).
2. The skill writes the spec to `<repo>/docs/specs/YYYY-MM-DD-<feature-slug>-spec.md`. Capture this path into orchestrator state as `SPEC_PATH`.
3. Do not proceed to Phase 2 until the brainstorming skill returns `approved`.
4. If the user aborts during brainstorming: stop cleanly, leave the partial spec in place, exit the orchestrator.
5. Context-cost note: if the spec exceeds ~5k tokens, before Phase 2 confirm you can re-read it from disk later rather than keeping it in your scrollback.

## Phase 2: Plan

Phase 2 runs in a subagent — no user interaction required.

1. Invoke `andrej-karpathy-skills:karpathy-guidelines` in THIS session to set behavioral baseline (required by `~/.claude/rules/common/development-workflow.md`).

2. Dispatch the `planner` subagent using the `Agent` tool with `subagent_type: general-purpose`:

```text
You are the planner subagent for /ship-feature. Read the approved spec at: {SPEC_PATH}

DEFAULT_BRANCH: {DEFAULT_BRANCH} (use this in concerns.json's "default_branch" field)

Use the `superpowers:writing-plans` skill to produce a TDD implementation plan, with one modification: decompose the feature into N INDEPENDENT CONCERNS. Each concern ships as its own branch, its own PR, its own CI cycle — no umbrella branch.

Deliverables (absolute paths, both written to disk):
1. Plan markdown at: {REPO_ROOT}/docs/plans/{YYYY-MM-DD}-{FEATURE_SLUG}-plan.md
2. concerns.json at: {REPO_ROOT}/docs/plans/{YYYY-MM-DD}-{FEATURE_SLUG}-concerns.json

Schema for concerns.json:

{
  "feature_slug": "<slug>",
  "default_branch": "<from orchestrator>",
  "concerns": [
    {
      "slug": "<kebab-case>",
      "branch": "feat/<feature_slug>-<slug>",
      "deps": ["<other-concern-slug>", ...],
      "tasks": [
        {"id": "t1", "desc": "<what>", "tests_first": true, "acceptance": "<concrete criteria>"}
      ],
      "pr_title": "<conventional-commit-style>"
    }
  ]
}

Rules:
- Concerns must be topologically sortable (no cycles). If you can't make them acyclic, return {"error": "circular_deps", "cycle": [...]} instead and do not write files.
- Each concern must be independently revertible: no two concerns may modify the same file. If two concerns both need to change a shared file (e.g., updating a shared constant), extract that change into a prerequisite concern that both depend on. If this is impossible, return {"error": "unsplittable_concern", "reason": "..."} and do not write files.
- Return ONLY this JSON (no prose): {"plan_path": "...", "concerns_path": "...", "concerns": <inline copy of concerns array>}
```

3. Parse the subagent's returned JSON. Store `PLAN_PATH`, `CONCERNS_PATH`, and the concerns array in orchestrator state.

4. Error handling on subagent return:
   - If `{"error": "circular_deps", ...}`: re-dispatch once with the cycle appended to the prompt: `"Previous attempt had a cycle: <cycle>. Restructure to eliminate it."`. If the second attempt also fails, bail to the user.
   - If `{"error": "unsplittable_concern", "reason": "..."}`: bail to the user immediately with the reason. Do NOT retry — the planner determined this is structurally impossible.

5. No user checkpoint. Proceed to Phase 3.

## Phase 3: Worktree per concern

Create one worktree per concern at `<repo-parent>/<repo-name>-<concern.slug>/`.

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
REPO_NAME=$(basename "$REPO_ROOT")
REPO_PARENT=$(dirname "$REPO_ROOT")
```

For each concern in the concerns array:

```bash
WORKTREE="$REPO_PARENT/$REPO_NAME-<concern.slug>"
BRANCH="<concern.branch>"

# Pre-check: branch must not already exist locally OR on origin (e.g., from a prior failed run)
if git show-ref --verify --quiet "refs/heads/$BRANCH"; then
  echo "FAIL Phase 3 pre-check: branch '$BRANCH' already exists locally."
  echo "FIX: from a previous failed run? Delete with: git branch -D '$BRANCH'"
  echo "(Or rename the concern slug to avoid collision.)"
  exit 1
fi
if git ls-remote --exit-code --heads origin "$BRANCH" >/dev/null 2>&1; then
  echo "FAIL Phase 3 pre-check: branch '$BRANCH' already exists on origin."
  echo "FIX: delete remote with: git push origin --delete '$BRANCH'"
  echo "(Or rename the concern slug to avoid collision.)"
  exit 1
fi

git worktree add -b "$BRANCH" "$WORKTREE" "origin/$DEFAULT_BRANCH"

# Worktree-cwd-guard (first pass) — must ALL pass
pushd "$WORKTREE" >/dev/null
HEAD_BRANCH=$(git rev-parse --abbrev-ref HEAD)
WT_ABS=$(pwd)
popd >/dev/null

# Guard 1: HEAD is not the default branch or main/master
if [ "$HEAD_BRANCH" = "$DEFAULT_BRANCH" ] || [ "$HEAD_BRANCH" = "main" ] || [ "$HEAD_BRANCH" = "master" ]; then
  echo "FAIL Phase 3 Guard 1: worktree HEAD is $HEAD_BRANCH"
  exit 1
fi

# Guard 2: worktree path is not the main repo
if [ "$WT_ABS" = "$REPO_ROOT" ]; then
  echo "FAIL Phase 3 Guard 2: worktree path equals main repo"
  exit 1
fi

# Guard 3: worktree path is not $HOME
if [ "$WT_ABS" = "$HOME" ]; then
  echo "FAIL Phase 3 Guard 3: worktree path equals \$HOME"
  exit 1
fi

# Guard 4: worktree is registered
git worktree list --porcelain | grep -qx "worktree $WT_ABS" || {
  echo "FAIL Phase 3 Guard 4: worktree not registered in git worktree list"
  exit 1
}

echo "Worktree for <concern.slug> OK: $WT_ABS on $HEAD_BRANCH"
```

On any guard failure: abort the entire command immediately. Do NOT attempt to recover automatically — worktree state is delicate and the user should investigate.

## Phase 4: Execute concerns

Execute concerns in parallel, capped at 3, respecting the dependency DAG.

**Scheduler algorithm:**

```text
ready   = [concerns with deps=[]]
pending = [concerns with deps != []]
running = []
done    = []
failed  = []
aborted = []

while ready or running:
  while len(running) < 3 and ready:
    c = ready.pop()
    dispatch concern-executor subagent for c (run_in_background=true)
    running.append(c)

  await any running subagent to finish:
    if success and PR merged (Phase 7 reached for this concern):
      done.append(c)
      for every concern whose deps ⊆ done: move from pending → ready
    else:
      failed.append(c)
      for every concern downstream of c (transitively): mark aborted

  if failed and no more running:
    break

report {done, failed, aborted}
```

Use `Agent(run_in_background=true)` to dispatch concerns in parallel. The orchestrator awaits completions via Claude Code's background-agent notification mechanism (you receive a notification message when each subagent finishes). Handle each result as it arrives, then continue scheduling.

**Concern-executor subagent prompt (Agent tool, subagent_type: general-purpose):**

```text
You are the concern-executor subagent for /ship-feature. Run in the worktree and drive a single concern from first commit to merged PR.

Inputs:
- CONCERN: {concern object from concerns.json}
- WORKTREE: {absolute path}
- SPEC_PATH: {absolute path to spec}
- DEFAULT_BRANCH: {default branch name}

Before anything:
- `cd "$WORKTREE"`
- Re-verify the four worktree-cwd-guards (these are inlined here because subagents do not have access to the orchestrator's design doc):
  1. `git rev-parse --abbrev-ref HEAD` is not in {`main`, `master`, $DEFAULT_BRANCH}
  2. `pwd` (the worktree path) ≠ the main repo path you arrived from
  3. `pwd` ≠ $HOME
  4. `pwd` appears in `git worktree list --porcelain` (formatted as `worktree <path>`)
- Abort if any guard fails.

For each task in CONCERN.tasks, in order:
  1. Use `superpowers:test-driven-development`: write failing test first, commit the test, implement until green.
  2. Use `superpowers:verification-before-completion`: run the full test suite, typecheck, lint. Capture output.
  3. Use `superpowers:requesting-code-review` to dispatch TWO reviewers in parallel:
     a. Spec-conformance reviewer: give it SPEC_PATH + this task's acceptance criteria + `git diff HEAD~1`. Ask "Does this satisfy the spec?"
     b. Quality reviewer: the `code-reviewer` agent on the diff.
  4. If EITHER reviewer returns CRITICAL or HIGH severity findings: fix them, commit, re-review. One "iteration" = one full pair of reviewers re-running on the latest diff. Max 2 iterations per task; after the 2nd iteration, if any CRITICAL or HIGH remains, return {"status": "failed", "task_id": ..., "findings": [...]} and STOP — do not continue with later tasks.
  5. Commit the task with a conventional-commits type (feat/fix/refactor/test/docs/chore).

After all tasks pass:
- Programmatic push guard (run this verbatim before pushing):
  ```
  for forbidden in main master "$DEFAULT_BRANCH"; do
    if [ "$BRANCH" = "$forbidden" ]; then
      echo "ERROR concern-executor: refusing to push '$BRANCH' (matches forbidden '$forbidden')"
      exit 1
    fi
  done
  ```
- Push the branch: `git push -u origin "$BRANCH"`
- Return: {"status": "ready_for_pr", "branch": "$BRANCH", "head_sha": "$(git rev-parse HEAD)", "commits": [{"sha": "<full sha>", "subject": "<first line>"}], "test_output": "<final test suite stdout+stderr, truncated to ~5000 chars>"}

NEVER push to $DEFAULT_BRANCH. NEVER force-push. NEVER skip hooks.

Return only the JSON result. No prose.
```

After a concern-executor returns `ready_for_pr`: hand off to Phase 5 for that concern. After Phase 5→6→7 complete for a concern, mark it `done` in the scheduler and continue.

If a concern-executor returns `failed` or crashes: mark failed, abort downstream concerns, preserve already-merged concerns, continue handling currently-running concerns until completion, then bail to the user with a structured report.

## Phase 5: PR per concern

Runs once per concern after its concern-executor returns `ready_for_pr`.

1. Invoke `superpowers:finishing-a-development-branch` to draft the PR body from the concern's commits + acceptance criteria. Capture as `PR_BODY_DRAFT`.

2. Dispatch the `verify-pr-body` subagent (Agent tool, subagent_type: general-purpose):

```text
You are the verify-pr-body subagent for /ship-feature. Your job is to catch fabricated facts in the PR body before the PR opens.

Inputs:
- PR_BODY_DRAFT: <the markdown>
- WORKTREE: <absolute path>
- DIFF: <output of `git diff origin/$DEFAULT_BRANCH...HEAD`>
- TEST_OUTPUT: <test_output field from concern-executor's ready_for_pr JSON>

For EVERY claim in PR_BODY_DRAFT:

1. File paths: for each path mentioned (e.g. `src/auth/login.ts`), run `test -e "$WORKTREE/<path>"`. Any missing path → problem.
2. Function/class/symbol names: for each symbol mentioned in prose (e.g. `validateSession()`, `class TokenStore`), Grep the worktree for its definition. Not found → problem.
3. CLI flags and config keys: for each flag/key mentioned, Grep the worktree. Not found → problem.
4. URLs: for each URL, WebFetch it. Non-2xx → problem.
5. Quoted log lines or CLI output: verify they appear verbatim in the test_output provided or can be produced by a command shown in the body.

Return ONLY: {"ok": true} OR {"ok": false, "problems": [{"claim": "...", "reason": "..."}]}
```

3. If `{"ok": true}`: set `PR_BODY_VERIFIED = PR_BODY_DRAFT` (or the rewritten body if step 3 looped) and proceed to step 4.
   If `{"ok": false}`: show the problems to the drafter (invoke finishing-a-development-branch again with the problem list), request a rewrite. The rewritten body REPLACES `PR_BODY_DRAFT` for the next verification pass. Max 1 rewrite (so 2 total verify-pr-body attempts: the initial pass + one re-verify after rewrite). If the second verify also returns `{"ok": false}` → pause the orchestrator and report the problems to the user.

4. **Never-push-to-main guard** before opening the PR. (`$BRANCH` is from the concern-executor's `ready_for_pr` JSON result; `$PR_BODY_VERIFIED` is the body that just passed verify-pr-body.)

```bash
for forbidden in main master "$DEFAULT_BRANCH"; do
  if [ "$BRANCH" = "$forbidden" ]; then
    echo "FAIL Phase 5 guard: concern branch is $BRANCH — would push to default."
    exit 1
  fi
done
```

5. Open the PR and capture the PR number:

```bash
PR_URL=$(gh pr create \
  --base "$DEFAULT_BRANCH" \
  --head "$BRANCH" \
  --title "<concern.pr_title>" \
  --body "$PR_BODY_VERIFIED")
PR_NUMBER=$(echo "$PR_URL" | grep -oE '[0-9]+$')
[ -n "$PR_NUMBER" ] || { echo "FAIL Phase 5: could not parse PR number from gh output: $PR_URL"; exit 1; }
```

`PR_NUMBER` is now in per-concern state for Phase 6 (CI loop) and Phase 7 (merge).

## Phase 6: CI recovery loop

Poll CI, auto-fix on failure, cap at 3 attempts.

```text
attempts = 0
while attempts < 3:
    # gh pr checks --watch waits for checks; --fail-fast returns as soon as one check fails (so we don't wait through the rest before fixing)
    result = Bash("gh pr checks $PR_NUMBER --watch --fail-fast --json name,state,conclusion")
    if all checks conclusion == SUCCESS:
        break
    attempts += 1
    failure_logs = Bash("gh run view --log-failed $(gh pr checks $PR_NUMBER --json name,state,databaseId --jq '.[] | select(.state==\"FAILURE\") | .databaseId' | head -1)")
    dispatch ci-fixer subagent with {PR_NUMBER, BRANCH, WORKTREE, failure_logs}
if attempts == 3 and still not all green:
    Bash("gh pr view $PR_NUMBER --json state,statusCheckRollup")
    pause; report to user; skip merge for this concern
```

**ci-fixer subagent prompt (Agent tool, subagent_type: general-purpose):**

```text
You are the ci-fixer subagent for /ship-feature. A CI run on PR #{PR_NUMBER} (branch {BRANCH}) failed.

Inputs:
- FAILURE_LOGS: <from gh run view --log-failed>
- WORKTREE: <absolute path>
- BRANCH: <branch name>

Process:
1. `cd "$WORKTREE"` and assert the four worktree-cwd-guards from Phase 3.
2. Use `superpowers:systematic-debugging` to diagnose the failure. Do NOT just make symptoms disappear — find the root cause.
3. Fix the issue. Prefer fixing implementation over weakening tests.
4. Run the same test/lint/typecheck commands locally that CI runs. Confirm they now pass.
5. Commit with a conventional `fix:` message that names what failed.
6. Programmatic push guard, then push:
   ```
   for forbidden in main master "$DEFAULT_BRANCH"; do
     if [ "$BRANCH" = "$forbidden" ]; then
       echo "ERROR ci-fixer: refusing to push '$BRANCH' (matches forbidden '$forbidden')"
       exit 1
     fi
   done
   git push origin "$BRANCH"
   ```
   (NO force, NO --no-verify.)

NEVER push to $DEFAULT_BRANCH. NEVER force-push. NEVER skip hooks.

Return: {"diagnosis": "...", "commit_sha": "..."} or {"status": "gave_up", "reason": "..."}
```

## Phase 7: Merge

Runs once the concern's PR is green.

```bash
gh pr merge "$PR_NUMBER" --squash --auto --delete-branch
```

Mark this concern `done` in the scheduler. Any concern whose `deps` now all resolve to `done` becomes eligible for dispatch (if scheduler capacity allows).

If `gh pr merge` fails (e.g., merge conflict with main that appeared after Phase 6): mark the concern failed, abort downstream concerns, continue other in-flight concerns, bail to user with the error.

## Phase 8: Cleanup

Runs after the scheduler terminates (all concerns done, failed, or aborted).

For each worktree path `W` created in Phase 3:

```bash
CWD=$(pwd)

# Worktree-cwd-guard (second pass) — ALL must pass for removal
# Guard A: we are not standing inside W
case "$CWD" in
  "$W"|"$W"/*)
    echo "SKIP cleanup: cwd $CWD is inside target worktree $W"
    continue
    ;;
esac

# Guard B: W is not the main repo
if [ "$W" = "$REPO_ROOT" ]; then
  echo "SKIP cleanup: target $W equals main repo root — refusing"
  continue
fi

# Guard C: W is not $HOME
if [ "$W" = "$HOME" ]; then
  echo "SKIP cleanup: target $W equals \$HOME — refusing"
  continue
fi

# Guard D: W is registered in git worktree list
if ! git worktree list --porcelain | grep -qx "worktree $W"; then
  echo "SKIP cleanup: target $W not in git worktree list — refusing"
  continue
fi

# All guards passed
git worktree remove "$W"   # NO --force
echo "Removed worktree $W"
```

Any skipped worktrees get logged but the orchestrator continues.

After all worktrees handled: invoke the `git-cleanup` skill against `$REPO_ROOT` to prune merged local branches and remote-tracking refs.

## Global error handling

At any phase, on unrecoverable failure:

1. Emit a structured report containing:
   - Phase reached
   - Concerns: `{done: [...], in_flight: [...], failed: [...], aborted: [...]}` (in this model `done` means "merged to default branch")
   - For each failed concern: the returned error, last commit SHA, PR number if any
   - Paths to spec, plan, concerns.json (these persist on disk)
   - Any worktrees that could not be cleaned up (listed with reason)

2. DO NOT attempt auto-recovery beyond the per-phase bounded retries (spec revision ≤ user choice, plan cycle ≤ 1 retry, review iterations ≤ 2 per task, fabrication rewrite ≤ 1 — i.e., 2 total verify-pr-body attempts, CI fix ≤ 3).

3. DO NOT delete the spec, plan, or any merged PRs — the user may want to resume or re-run.

4. Exit.

Resumption is currently manual: the user re-invokes `/ship-feature` (which will re-brainstorm a fresh spec) OR resumes from the existing spec/plan by reading them and driving the remaining concerns manually. A `--resume` flag is out of scope for v1.
