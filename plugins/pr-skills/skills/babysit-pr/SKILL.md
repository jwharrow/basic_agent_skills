---
name: babysit-pr
description: >-
  Autonomously monitors a PR until it is ready to merge. Loops up to 5 times:
  checks CI status, fixes failing checks locally, addresses straightforward review
  comments, resolves merge conflicts, and pushes. Stops and reports when the PR
  is green and review-clean, or when it hits something that needs human judgment.
  Use when someone says "babysit this PR", "watch this PR", or "get this PR ready to merge".
disable-model-invocation: true
---

# Babysit PR

## Input

The user provides a GitHub PR URL or number as an argument or in conversation (e.g. `/babysit-pr 123` or `/babysit-pr https://github.com/org/repo/pull/123`).

If `$ARGUMENTS` is provided, treat it as the PR URL or number.

If neither is provided, detect the PR from the current branch:
```bash
gh pr view --json number,url,headRefName,state
```

If no PR is found on the current branch, ask: "What's the PR URL or number?"

---

## Prerequisites

- `gh auth status` — must be authenticated to GitHub
- Must be run from inside a git repository

If `gh` is not authenticated, stop and tell the user: "Run `gh auth login` first."

---

## Setup

```bash
# Detect default branch
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef -q '.defaultBranchRef.name')

# Check out the PR branch
gh pr checkout {pr_number}
BRANCH=$(git rev-parse --abbrev-ref HEAD)
```

---

## Main Loop

Run the following loop until the PR is ready or the max iteration limit is reached.

**Max iterations: 5.** After 5 rounds without resolution, stop and report remaining blockers.

---

### Each Iteration

#### 1. Assess current state

Gather all signals in parallel:

```bash
# CI check results
gh pr checks {pr_number}

# PR merge state
gh pr view {pr_number} --json mergeStateStatus,mergeable,reviewDecision

# Unresolved review threads (GraphQL)
gh api graphql -f query='
{
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {pr_number}) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          isOutdated
          comments(first: 3) {
            nodes { body, author { login }, path, line }
          }
        }
      }
    }
  }
}'
```

Classify the state:
- **Ready**: all CI checks passing, no unresolved threads, no merge conflicts → done
- **CI failing**: one or more checks failing
- **Unresolved comments**: reviewer threads that are not resolved or outdated
- **Merge conflict**: `mergeStateStatus` is `DIRTY` or `CONFLICTING`
- **Changes requested**: `reviewDecision` is `CHANGES_REQUESTED`

#### 2. Triage and fix — in priority order

**Priority 1: CI failures**

```bash
# Get the run IDs for failing checks
gh run list --branch "$BRANCH" --status failure --json databaseId,name

# Get logs for a failing run
gh run view {run_id} --log-failed
```

For each failing check:
- **Test failures**: Read the test output, find the failing assertion, fix the code or test
- **Lint failures**: Run the linter locally, apply fixes
- **Build failures**: Read the error, fix the source
- **Type errors**: Fix type issues in the flagged files

After fixing, commit and push:
```bash
git add -A
git commit -m "fix: address CI failure in {check_name}"
git push origin "$BRANCH"
```

**Priority 2: Merge conflicts**

```bash
git fetch origin "$DEFAULT_BRANCH"
git rebase "origin/$DEFAULT_BRANCH"
# Resolve conflicts, then:
git push --force-with-lease origin "$BRANCH"
```

Only rebase if there are actual conflicts. Do not force-push unnecessarily.

**Priority 3: Unresolved review comments**

For each unresolved, non-outdated thread:

1. Read the comment and classify:
   - **Clear, actionable code change** (e.g. "add null check", "rename variable", "extract to helper") → make the change
   - **Subjective or architectural** (e.g. "consider a different pattern", "should this be a separate service?") → **stop and flag for user**
   - **A question needing clarification** → **stop and flag for user**

2. For actionable comments: make the change, commit, push, and note it in the iteration report
3. For ambiguous comments: collect them and surface to the user at the end of the iteration

```bash
git add -A
git commit -m "fix: address review comment from @{reviewer} on {file}:{line}"
git push origin "$BRANCH"
```

#### 3. Iteration report

After each iteration, report briefly:
```
Iteration N/5:
  CI: {passing N/M | fixed: X | still failing: Y}
  Comments: {resolved N | flagged for you: M}
  Conflicts: {none | resolved}
  Action: pushed commit {sha}
```

---

## Completion

**Done (ready to merge):**
```
PR #{pr_number} is ready to merge.
  - All CI checks passing
  - No unresolved review threads
  - No merge conflicts
  - Branch is up to date with {DEFAULT_BRANCH}
```

**Stopped — needs your input:**
```
PR #{pr_number} — stopped after N iterations. Remaining issues need your judgment:

  Ambiguous comments:
    [1] @reviewer — file.ts:42: "Consider a different architecture here..."
        → Not addressed (architectural decision, needs your call)

  Still failing CI:
    [1] integration-tests — flaky test in test/api.test.ts:88, may be pre-existing
```

**Max iterations reached:**
```
Reached 5 iterations. Remaining blockers:
  [list all unresolved items]
```

---

## Rules

- **Never merge** — just get the PR to a mergeable state. The user decides when to merge.
- **Never skip pre-commit hooks** (no `--no-verify`)
- **Never force-push to the default branch**
- **Never dismiss reviews** — only address the feedback
- **Never use `git push --force`** — use `git push --force-with-lease` after rebasing
- **Create new commits** for fixes rather than amending, unless the PR has only one commit and amending is clearly cleaner
- If a CI failure also fails on the default branch (pre-existing), note it and move on — don't try to fix it
- If a review comment is subjective, architectural, or requires user context, flag it rather than guessing
