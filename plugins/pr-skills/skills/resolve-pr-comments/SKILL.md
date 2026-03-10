---
name: resolve-pr-comments
description: >-
  Given a PR link, fetches all review comments, determines which are unaddressed,
  and proposes code changes or replies for each — conversationally. When the user
  confirms the plan, makes the changes, merges latest from main, re-runs all checks,
  and pushes. Use when the user wants to work through PR review feedback.
disable-model-invocation: true
---

# Resolve PR Comments

## Input

The user provides a GitHub PR URL as an argument or in conversation (e.g. `/resolve-pr-comments https://github.com/org/repo/pull/123`).

If `$ARGUMENTS` is provided, treat it as the PR URL.

If nothing is provided, ask: "What's the PR URL?"

Parse `{owner}`, `{repo}`, and `{pr_number}` from the URL.

---

## Prerequisites

Before starting, verify:
- `gh auth status` — must be authenticated to GitHub
- Must be run from inside a git repository

If `gh` is not authenticated, stop and tell the user: "Run `gh auth login` first."

---

## Step 1: Detect the default branch

```bash
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef -q '.defaultBranchRef.name')
```

---

## Step 2: Check out the PR branch

Before fetching comments or making any changes, switch to the PR's head branch:

```bash
gh pr checkout {pr_number}
```

This ensures all subsequent code changes land on the correct branch. Confirm which branch you're now on:

```bash
git rev-parse --abbrev-ref HEAD
```

---

## Step 3: Fetch all comments

```bash
# Inline review thread comments (attached to specific lines of code)
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
  --paginate \
  --jq '.[] | {id, path, line, body, user: .user.login, created_at, in_reply_to_id}'

# Review summaries (top-level review bodies, not inline)
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
  --paginate \
  --jq '.[] | {id, state, body, user: .user.login, submitted_at}'

# Conversation tab comments (general issue comments, not review comments)
gh api repos/{owner}/{repo}/issues/{pr_number}/comments \
  --paginate \
  --jq '.[] | {id, body, user: .user.login, created_at}'

# PR metadata
gh pr view {pr_number} --json title,body,headRefName,baseRefName,files

# Current diff
gh pr diff {pr_number}
```

Note the type of each comment — this matters for how you reply (Step 6):
- **Review comment** (`pulls/{pr_number}/comments`) — attached to a specific file/line
- **Review body** (`pulls/{pr_number}/reviews`) — top-level review summary
- **Issue comment** (`issues/{pr_number}/comments`) — general conversation comment

---

## Step 4: Triage — addressed vs unaddressed

For each review thread, use the GitHub GraphQL API to check resolution status:

```bash
gh api graphql -f query='
{
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {pr_number}) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          isOutdated
          comments(first: 5) {
            nodes {
              id
              body
              author { login }
              path
              line
            }
          }
        }
      }
    }
  }
}'
```

**Addressed** if any of:
- `isResolved: true` on the thread
- `isOutdated: true` (the code at that location changed enough to make the comment stale)
- The thread has a reply from the PR author that acknowledges and closes it

**Unaddressed** if:
- `isResolved: false` and the code at the referenced location is unchanged
- The reviewer explicitly re-raised the concern

For issue/conversation comments (not part of a review thread), check whether the PR author has replied.

Present a summary:
```
Found N comments:
  - X resolved / outdated
  - Y unaddressed (listed below)
```

---

## Step 5: Propose a response for each unaddressed comment

For each unaddressed comment, classify and propose:

```
[N] @reviewer — file.ts:42  (review comment)
Comment: "This should handle the null case before calling .map()"

Proposed action: Code change
  → Add a null guard before the `.map()` call on line 42

Proposed reply (optional):
  "Good catch — added a null guard before the `.map()` call."
```

**Action types:**
- **Code change**: modify the code to address the concern
- **Reply only**: the code is fine but needs clarification or context
- **Push back**: you disagree — include a proposed reply explaining why

Ask the user at the start: "Should I go through these one at a time, or show you all at once?"

---

## Step 6: Conversation

For each proposed action, the user can:
- **Approve** ("yes", "looks good", "do it")
- **Modify** ("change the reply to say X instead", "skip the code change, just reply")
- **Skip** ("ignore that one")
- **Ask a question** — answer it and re-propose

Keep a running list of approved actions. Do not make any changes yet.

When all comments have been reviewed, present the final plan and wait for confirmation:

```
Ready to execute:
  Code changes (N): ...
  Replies to post (N): ...
  Skipped (N): ...

Proceed?
```

---

## Step 7: Execute

### Make code changes

Apply each approved code change. Match surrounding code conventions.

### Post replies

Use the correct endpoint based on comment type:

**Review comment** (inline, attached to a file/line):
```bash
gh api repos/{owner}/{repo}/pulls/comments/{comment_id}/replies \
  -X POST \
  -f body="Your reply text"
```

**Review body** (top-level review summary) — reply as an issue comment:
```bash
gh api repos/{owner}/{repo}/issues/{pr_number}/comments \
  -X POST \
  -f body="Your reply text"
```

**Issue/conversation comment** — reply as an issue comment:
```bash
gh api repos/{owner}/{repo}/issues/{pr_number}/comments \
  -X POST \
  -f body="Your reply text"
```

### Merge latest from default branch

```bash
git fetch origin "$DEFAULT_BRANCH"
git merge "origin/$DEFAULT_BRANCH"
```

If there are merge conflicts, resolve them and commit. Tell the user what was conflicted.

### Run all checks

Discover and run the full check suite (look in `AGENTS.md`, `package.json`, `Makefile`, etc.). Fix any failures before pushing.

### Push safely

Before pushing, verify no merged/closed PR exists on this branch:

```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
gh pr list --head "$BRANCH" --state merged --json number,title
gh pr list --head "$BRANCH" --state closed --json number,title
```

If a merged/closed PR exists, create a new branch and push from there.

Push:
```bash
git push origin "$BRANCH"
```

After pushing, re-check PR state:
```bash
gh pr list --head "$BRANCH" --state merged --json number,title
```

Never use `git push --force`. Use `git push --force-with-lease` if a force push is needed after rebasing.

---

## Step 8: Report

```
Done.
  - Made N code changes
  - Posted N replies
  - Merged latest from {DEFAULT_BRANCH} (no conflicts / N conflicts resolved)
  - All checks passed
  - Pushed to origin/{branch}

PR: {pr_url}
```

---

## Notes

- Never post a reply without user approval
- Never push with failing checks
- The proposals are starting points — the user's preference always wins
- Never use `git push --force` — always use `git push --force-with-lease`
