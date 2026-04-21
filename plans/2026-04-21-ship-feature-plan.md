# /ship-feature Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scaffold a single personal slash command at `~/.claude/commands/ship-feature.md` that orchestrates the 8-phase ship-feature flow described in `~/.claude/docs/ship-feature-design.md`.

**Architecture:** One markdown file with YAML frontmatter and an orchestration prompt body. Subagent prompts are inlined as fenced code blocks. Verification is read-back grep assertions per task; a final smoke test runs `/ship-feature` against a sandbox repo.

**Tech Stack:** Claude Code slash command format (markdown + YAML frontmatter), `gh` CLI, `git` CLI, subagent dispatch via the `Agent` tool, skills: `superpowers:brainstorming`, `superpowers:using-git-worktrees`, `superpowers:test-driven-development`, `superpowers:verification-before-completion`, `superpowers:requesting-code-review`, `superpowers:finishing-a-development-branch`, `git-cleanup`, `andrej-karpathy-skills:karpathy-guidelines`.

**Note on commits:** `~/.claude/` is not a git repo, so the "commit" step after every task is a no-op here. Each task ends with a read-back verification instead.

---

### Task 1: Create the command file skeleton

**Files:**
- Create: `~/.claude/commands/ship-feature.md`

- [ ] **Step 1: Write the failing assertion (grep check)**

```bash
test ! -f ~/.claude/commands/ship-feature.md && echo "NOT EXISTS (expected)"
```
Expected: prints `NOT EXISTS (expected)`

- [ ] **Step 2: Write the skeleton**

```markdown
---
description: Ship a feature end-to-end — spec, plan, implement, PR, merge, cleanup — with one user checkpoint at spec approval
argument-hint: <one-line feature description>
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, Agent, Skill, AskUserQuestion, WebFetch, TaskCreate, TaskUpdate
---

You are running the `/ship-feature` orchestrator. The user's one-line feature description is:

$ARGUMENTS

Follow the design at `~/.claude/docs/ship-feature-design.md`. Execute phases in order. Create a TaskCreate todo for every phase so the user can see progress.

## Phase 0: Preconditions

<PLACEHOLDER>

## Phase 1: Spec

<PLACEHOLDER>

## Phase 2: Plan

<PLACEHOLDER>

## Phase 3: Worktree per concern

<PLACEHOLDER>

## Phase 4: Execute concerns

<PLACEHOLDER>

## Phase 5: PR per concern

<PLACEHOLDER>

## Phase 6: CI recovery loop

<PLACEHOLDER>

## Phase 7: Merge

<PLACEHOLDER>

## Phase 8: Cleanup

<PLACEHOLDER>

## Global error handling

<PLACEHOLDER>
```

Write this exact content to `~/.claude/commands/ship-feature.md`.

- [ ] **Step 3: Verify the skeleton**

```bash
grep -c "<PLACEHOLDER>" ~/.claude/commands/ship-feature.md
```
Expected: `10`

```bash
head -5 ~/.claude/commands/ship-feature.md
```
Expected: the frontmatter with `description:` and `argument-hint:`.

---

### Task 2: Phase 0 — Preconditions block

**Files:**
- Modify: `~/.claude/commands/ship-feature.md` — replace the first `<PLACEHOLDER>` (under `## Phase 0: Preconditions`)

- [ ] **Step 1: Write the assertion**

```bash
grep -q "git status --porcelain" ~/.claude/commands/ship-feature.md && echo "PASS" || echo "FAIL"
```
Expected: `FAIL`

- [ ] **Step 2: Replace the Phase 0 placeholder with**

```markdown
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
git fetch origin "$DEFAULT_BRANCH" --quiet
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
```

- [ ] **Step 3: Verify**

```bash
grep -c "DEFAULT_BRANCH" ~/.claude/commands/ship-feature.md
```
Expected: `≥ 6` (it's referenced in multiple fix messages and asserts).

```bash
grep -c "<PLACEHOLDER>" ~/.claude/commands/ship-feature.md
```
Expected: `9` (Phase 0 replaced, 9 remain).

---

### Task 3: Phase 1 — Invoke brainstorming skill

**Files:**
- Modify: `~/.claude/commands/ship-feature.md` — replace `<PLACEHOLDER>` under `## Phase 1: Spec`

- [ ] **Step 1: Write the assertion**

```bash
grep -q "superpowers:brainstorming" ~/.claude/commands/ship-feature.md && echo "PASS" || echo "FAIL"
```
Expected: `FAIL`

- [ ] **Step 2: Replace the Phase 1 placeholder with**

```markdown
Phase 1 runs in THIS session (not a subagent) so brainstorming can ask the user clarifying questions. The brainstorming skill's own User Review Gate is the Phase 1 approval checkpoint — do not add a second `AskUserQuestion`.

1. Invoke the `superpowers:brainstorming` skill with the user's one-liner ($ARGUMENTS) as seed material. Let the skill run its full native flow (explore repo → ask clarifying questions one-at-a-time → propose 2–3 approaches → present design section-by-section → write spec → User Review Gate).
2. The skill writes the spec to `<repo>/docs/specs/YYYY-MM-DD-<feature-slug>-spec.md`. Capture this path into orchestrator state as `SPEC_PATH`.
3. Do not proceed to Phase 2 until the brainstorming skill returns `approved`.
4. If the user aborts during brainstorming: stop cleanly, leave the partial spec in place, exit the orchestrator.
5. Context-cost note: if the spec exceeds ~5k tokens, before Phase 2 confirm you can re-read it from disk later rather than keeping it in your scrollback.
```

- [ ] **Step 3: Verify**

```bash
grep -q "superpowers:brainstorming" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -q "SPEC_PATH" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -c "<PLACEHOLDER>" ~/.claude/commands/ship-feature.md
```
Expected: `PASS`, `PASS`, `8`.

---

### Task 4: Phase 2 — Planner subagent prompt + dispatch

**Files:**
- Modify: `~/.claude/commands/ship-feature.md` — replace `<PLACEHOLDER>` under `## Phase 2: Plan`

- [ ] **Step 1: Write the assertion**

```bash
grep -q "concerns.json" ~/.claude/commands/ship-feature.md && echo "PASS" || echo "FAIL"
```
Expected: `FAIL`

- [ ] **Step 2: Replace the Phase 2 placeholder with**

````markdown
Phase 2 runs in a subagent — no user interaction required.

1. Invoke `andrej-karpathy-skills:karpathy-guidelines` in THIS session to set behavioral baseline (required by `~/.claude/rules/common/development-workflow.md`).

2. Dispatch the `planner` subagent using the `Agent` tool with `subagent_type: general-purpose`:

```text
You are the planner subagent for /ship-feature. Read the approved spec at: {SPEC_PATH}

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
- Each concern must be independently revertible — no cross-concern file ownership.
- Return ONLY this JSON (no prose): {"plan_path": "...", "concerns_path": "...", "concerns": <inline copy of concerns array>}
```

3. Parse the subagent's returned JSON. Store `PLAN_PATH`, `CONCERNS_PATH`, and the concerns array in orchestrator state.

4. If the subagent returned `{"error": "circular_deps", ...}`: re-dispatch once with the cycle appended to the prompt: `"Previous attempt had a cycle: <cycle>. Restructure to eliminate it."`. If the second attempt also fails, bail to the user.

5. No user checkpoint. Proceed to Phase 3.
````

- [ ] **Step 3: Verify**

```bash
grep -q "concerns.json" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -q "karpathy-guidelines" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -q "circular_deps" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -c "<PLACEHOLDER>" ~/.claude/commands/ship-feature.md
```
Expected: `PASS`, `PASS`, `PASS`, `7`.

---

### Task 5: Phase 3 — Worktree creation + guards

**Files:**
- Modify: `~/.claude/commands/ship-feature.md` — replace `<PLACEHOLDER>` under `## Phase 3: Worktree per concern`

- [ ] **Step 1: Write the assertion**

```bash
grep -q "git worktree list --porcelain" ~/.claude/commands/ship-feature.md && echo "PASS" || echo "FAIL"
```
Expected: `FAIL`

- [ ] **Step 2: Replace the Phase 3 placeholder with**

````markdown
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
````

- [ ] **Step 3: Verify**

```bash
grep -q "git worktree list --porcelain" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -c "Guard [1-4]" ~/.claude/commands/ship-feature.md
grep -c "<PLACEHOLDER>" ~/.claude/commands/ship-feature.md
```
Expected: `PASS`, `4`, `6`.

---

### Task 6: Phase 4 — DAG scheduler + concern-executor subagent

**Files:**
- Modify: `~/.claude/commands/ship-feature.md` — replace `<PLACEHOLDER>` under `## Phase 4: Execute concerns`

- [ ] **Step 1: Write the assertion**

```bash
grep -q "topologically" ~/.claude/commands/ship-feature.md && echo "PASS" || echo "FAIL"
```
Expected: `FAIL`

- [ ] **Step 2: Replace the Phase 4 placeholder with**

````markdown
Execute concerns in parallel, capped at 3, respecting the dependency DAG.

**Scheduler algorithm:**

```text
ready   = [concerns with deps=[]]
running = []
done    = []
failed  = []

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

Use `Agent(run_in_background=true)` to dispatch concerns in parallel. When a concern-executor returns, handle its result, then continue scheduling.

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
- Re-verify all four worktree-cwd-guards (see Phase 3 in the design doc). Abort if any fails.

For each task in CONCERN.tasks, in order:
  1. Use `superpowers:test-driven-development`: write failing test first, commit the test, implement until green.
  2. Use `superpowers:verification-before-completion`: run the full test suite, typecheck, lint. Capture output.
  3. Use `superpowers:requesting-code-review` to dispatch TWO reviewers in parallel:
     a. Spec-conformance reviewer: give it SPEC_PATH + this task's acceptance criteria + `git diff HEAD~1`. Ask "Does this satisfy the spec?"
     b. Quality reviewer: the `code-reviewer` agent on the diff.
  4. If EITHER reviewer returns CRITICAL or HIGH severity findings: fix them, commit, re-review. Max 2 review iterations per task. After 2 failed iterations, return {"status": "failed", "task_id": ..., "findings": [...]} and STOP — do not continue with later tasks.
  5. Commit the task with a conventional-commits type (feat/fix/refactor/test/docs/chore).

After all tasks pass:
- Push the branch: `git push -u origin "$BRANCH"`
- Return: {"status": "ready_for_pr", "branch": "$BRANCH", "head_sha": "$(git rev-parse HEAD)", "commits": [...], "test_output": "..."}

NEVER push to $DEFAULT_BRANCH. NEVER force-push. NEVER skip hooks.

Return only the JSON result. No prose.
```

After a concern-executor returns `ready_for_pr`: hand off to Phase 5 for that concern. After Phase 5→6→7 complete for a concern, mark it `done` in the scheduler and continue.

If a concern-executor returns `failed` or crashes: mark failed, abort downstream concerns, preserve already-merged concerns, continue handling currently-running concerns until completion, then bail to the user with a structured report.
````

- [ ] **Step 3: Verify**

```bash
grep -q "topologically\|scheduler" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -q "Max 2 review iterations" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -q "NEVER push to \$DEFAULT_BRANCH" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -c "<PLACEHOLDER>" ~/.claude/commands/ship-feature.md
```
Expected: `PASS`, `PASS`, `PASS`, `5`.

---

### Task 7: Phase 5 — PR drafting + fabrication check

**Files:**
- Modify: `~/.claude/commands/ship-feature.md` — replace `<PLACEHOLDER>` under `## Phase 5: PR per concern`

- [ ] **Step 1: Write the assertion**

```bash
grep -q "verify-pr-body" ~/.claude/commands/ship-feature.md && echo "PASS" || echo "FAIL"
```
Expected: `FAIL`

- [ ] **Step 2: Replace the Phase 5 placeholder with**

````markdown
Runs once per concern after its concern-executor returns `ready_for_pr`.

1. Invoke `superpowers:finishing-a-development-branch` to draft the PR body from the concern's commits + acceptance criteria. Capture as `PR_BODY_DRAFT`.

2. Dispatch the `verify-pr-body` subagent (Agent tool, subagent_type: general-purpose):

```text
You are the verify-pr-body subagent for /ship-feature. Your job is to catch fabricated facts in the PR body before the PR opens.

Inputs:
- PR_BODY_DRAFT: <the markdown>
- WORKTREE: <absolute path>
- DIFF: <output of `git diff origin/$DEFAULT_BRANCH...HEAD`>

For EVERY claim in PR_BODY_DRAFT:

1. File paths: for each path mentioned (e.g. `src/auth/login.ts`), run `test -e "$WORKTREE/<path>"`. Any missing path → problem.
2. Function/class/symbol names: for each symbol mentioned in prose (e.g. `validateSession()`, `class TokenStore`), Grep the worktree for its definition. Not found → problem.
3. CLI flags and config keys: for each flag/key mentioned, Grep the worktree. Not found → problem.
4. URLs: for each URL, WebFetch it. Non-2xx → problem.
5. Quoted log lines or CLI output: verify they appear verbatim in the test_output provided or can be produced by a command shown in the body.

Return ONLY: {"ok": true} OR {"ok": false, "problems": [{"claim": "...", "reason": "..."}]}
```

3. If `{"ok": true}`: proceed to step 4.
   If `{"ok": false}`: show the problems to the drafter (invoke finishing-a-development-branch again with the problem list), request a rewrite. Max 2 rewrite attempts. Still failing → pause the orchestrator and report the problems to the user.

4. **Never-push-to-main guard** before opening the PR:

```bash
for forbidden in main master "$DEFAULT_BRANCH"; do
  if [ "$BRANCH" = "$forbidden" ]; then
    echo "FAIL Phase 5 guard: concern branch is $BRANCH — would push to default."
    exit 1
  fi
done
```

5. Open the PR:

```bash
gh pr create \
  --base "$DEFAULT_BRANCH" \
  --head "$BRANCH" \
  --title "<concern.pr_title>" \
  --body "$PR_BODY_VERIFIED"
```

Capture the returned PR number as `PR_NUMBER` in the per-concern state.
````

- [ ] **Step 3: Verify**

```bash
grep -q "verify-pr-body" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -q "Never-push-to-main guard" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -q "gh pr create" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -c "<PLACEHOLDER>" ~/.claude/commands/ship-feature.md
```
Expected: `PASS`, `PASS`, `PASS`, `4`.

---

### Task 8: Phase 6 — CI recovery loop

**Files:**
- Modify: `~/.claude/commands/ship-feature.md` — replace `<PLACEHOLDER>` under `## Phase 6: CI recovery loop`

- [ ] **Step 1: Write the assertion**

```bash
grep -q "ci-fixer" ~/.claude/commands/ship-feature.md && echo "PASS" || echo "FAIL"
```
Expected: `FAIL`

- [ ] **Step 2: Replace the Phase 6 placeholder with**

````markdown
Poll CI, auto-fix on failure, cap at 3 attempts.

```text
attempts = 0
while attempts < 3:
    # gh pr checks blocks until all checks complete
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
6. `git push origin "$BRANCH"` (NO force, NO --no-verify).

NEVER push to $DEFAULT_BRANCH. NEVER force-push. NEVER skip hooks.

Return: {"diagnosis": "...", "commit_sha": "..."} or {"status": "gave_up", "reason": "..."}
```
````

- [ ] **Step 3: Verify**

```bash
grep -q "ci-fixer" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -q "attempts < 3" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -q "systematic-debugging" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -c "<PLACEHOLDER>" ~/.claude/commands/ship-feature.md
```
Expected: `PASS`, `PASS`, `PASS`, `3`.

---

### Task 9: Phase 7 — Merge

**Files:**
- Modify: `~/.claude/commands/ship-feature.md` — replace `<PLACEHOLDER>` under `## Phase 7: Merge`

- [ ] **Step 1: Write the assertion**

```bash
grep -q "gh pr merge" ~/.claude/commands/ship-feature.md && echo "PASS" || echo "FAIL"
```
Expected: `FAIL`

- [ ] **Step 2: Replace the Phase 7 placeholder with**

```markdown
Runs once the concern's PR is green.

```bash
gh pr merge "$PR_NUMBER" --squash --auto --delete-branch
```

Mark this concern `done` in the scheduler. Any concern whose `deps` now all resolve to `done` becomes eligible for dispatch (if scheduler capacity allows).

If `gh pr merge` fails (e.g., merge conflict with main that appeared after Phase 6): mark the concern failed, abort downstream concerns, continue other in-flight concerns, bail to user with the error.
```

- [ ] **Step 3: Verify**

```bash
grep -q "gh pr merge .* --squash --auto --delete-branch" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -c "<PLACEHOLDER>" ~/.claude/commands/ship-feature.md
```
Expected: `PASS`, `2`.

---

### Task 10: Phase 8 — Cleanup with worktree-cwd-guard

**Files:**
- Modify: `~/.claude/commands/ship-feature.md` — replace `<PLACEHOLDER>` under `## Phase 8: Cleanup`

- [ ] **Step 1: Write the assertion**

```bash
grep -c "worktree-cwd-guard" ~/.claude/commands/ship-feature.md
```
Expected: some count. Note it; it should be higher after this task.

- [ ] **Step 2: Replace the Phase 8 placeholder with**

````markdown
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
````

- [ ] **Step 3: Verify**

```bash
grep -c "Guard [A-D]" ~/.claude/commands/ship-feature.md
grep -q "NO --force" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -q "git-cleanup" ~/.claude/commands/ship-feature.md && echo "PASS"
grep -c "<PLACEHOLDER>" ~/.claude/commands/ship-feature.md
```
Expected: `4`, `PASS`, `PASS`, `1`.

---

### Task 11: Global error handling section

**Files:**
- Modify: `~/.claude/commands/ship-feature.md` — replace the final `<PLACEHOLDER>` under `## Global error handling`

- [ ] **Step 1: Write the assertion**

```bash
grep -c "<PLACEHOLDER>" ~/.claude/commands/ship-feature.md
```
Expected: `1`.

- [ ] **Step 2: Replace the Phase's final placeholder with**

```markdown
At any phase, on unrecoverable failure:

1. Emit a structured report containing:
   - Phase reached
   - Concerns: `{done: [...], merged: [...], in_flight: [...], failed: [...], aborted: [...]}`
   - For each failed concern: the returned error, last commit SHA, PR number if any
   - Paths to spec, plan, concerns.json (these persist on disk)
   - Any worktrees that could not be cleaned up (listed with reason)

2. DO NOT attempt auto-recovery beyond the per-phase bounded retries (spec revision ≤ user choice, plan cycle ≤ 1 retry, review iterations ≤ 2 per task, fabrication rewrite ≤ 2, CI fix ≤ 3).

3. DO NOT delete the spec, plan, or any merged PRs — the user may want to resume or re-run.

4. Exit.

Resumption is currently manual: the user re-invokes `/ship-feature` (which will re-brainstorm a fresh spec) OR resumes from the existing spec/plan by reading them and driving the remaining concerns manually. A `--resume` flag is out of scope for v1.
```

- [ ] **Step 3: Verify**

```bash
grep -c "<PLACEHOLDER>" ~/.claude/commands/ship-feature.md
```
Expected: `0`.

---

### Task 12: End-to-end smoke test

**Files:**
- Read: `~/.claude/commands/ship-feature.md` (full file)
- Test target: a disposable sandbox repo

- [ ] **Step 1: Final read-back — sanity check the whole file**

Read `~/.claude/commands/ship-feature.md` end-to-end. Verify:

- Frontmatter has `description`, `argument-hint`, `allowed-tools`
- No remaining `<PLACEHOLDER>` strings
- All eight phase headings present
- All four friction-point mitigations visible (search for: "never push", "concerns.json", "Guard [A-D]", "verify-pr-body")

- [ ] **Step 2: Create a sandbox repo for smoke testing**

```bash
mkdir -p ~/tmp/ship-feature-smoke && cd ~/tmp/ship-feature-smoke
git init
echo "# sandbox" > README.md
git add README.md
git commit -m "init"
gh repo create ship-feature-smoke --private --source=. --push
```

Expected: repo created, default branch pushed.

- [ ] **Step 3: Invoke `/ship-feature` against the sandbox**

In the sandbox repo dir, invoke:
```
/ship-feature add a hello command that prints the current date
```

Expected behavior:
- Phase 0 preconditions pass (clean repo, on default branch, authenticated).
- Phase 1 starts: brainstorming skill asks clarifying questions.
- At the User Review Gate, user can abort cleanly.

Do NOT let the smoke test run to merge — abort at the spec approval gate. The goal here is to prove the orchestrator reaches Phase 1 with the right state.

- [ ] **Step 4: Teardown**

```bash
gh repo delete ship-feature-smoke --confirm
rm -rf ~/tmp/ship-feature-smoke
```

---

## Self-Review

**1. Spec coverage:**

| Design doc section | Task in plan |
|---|---|
| Phase 0 Preconditions | Task 2 |
| Phase 1 Spec (in-session brainstorming) | Task 3 |
| Phase 2 Plan (planner subagent, concerns.json) | Task 4 |
| Phase 3 Worktree + 4 guards | Task 5 |
| Phase 4 DAG scheduler + concern-executor | Task 6 |
| Phase 5 PR + verify-pr-body + never-push-to-main | Task 7 |
| Phase 6 CI recovery loop + ci-fixer | Task 8 |
| Phase 7 Merge | Task 9 |
| Phase 8 Cleanup + second-pass guards | Task 10 |
| Global failure modes | Task 11 |
| End-to-end verification | Task 12 |

All design sections covered.

**2. Placeholder scan:** No "TBD" / "similar to" / "add appropriate error handling" / "handle edge cases" / "implement later" in the plan. Every task shows the exact replacement content. ✓

**3. Type consistency:** Variable names used across tasks are consistent:
- `DEFAULT_BRANCH` (Tasks 2, 3, 4, 5, 6, 7, 8, 10)
- `SPEC_PATH` (Tasks 3, 4, 6)
- `PLAN_PATH` / `CONCERNS_PATH` (Task 4)
- `REPO_ROOT` (Tasks 5, 10)
- `WORKTREE` / `WT_ABS` (Tasks 5, 6, 8, 10)
- `BRANCH` (Tasks 5, 6, 7, 8, 10)
- `PR_NUMBER` (Tasks 7, 8, 9)
- `PR_BODY_DRAFT` / `PR_BODY_VERIFIED` (Task 7)
- Worktree-cwd-guard numbered 1–4 in Phase 3 (Task 5), lettered A–D in Phase 8 (Task 10). Intentional: Phase 3 checks pre-write state, Phase 8 checks pre-remove state, the letter/number difference flags them as separate passes.

All consistent. ✓

---

## Execution Handoff

Plan complete and saved to `~/.claude/plans/2026-04-21-ship-feature-plan.md`. Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task (Tasks 1–11), review between tasks, fast iteration. Task 12 (smoke test) runs inline since it needs real gh/git interaction.

**2. Inline Execution** — Execute tasks in this session using `superpowers:executing-plans`, batch execution with checkpoints for review.

Which approach?
