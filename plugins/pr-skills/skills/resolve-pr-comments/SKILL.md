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

The user provides a GitHub PR URL (e.g. `https://github.com/org/repo/pull/123`).

If no URL is provided, ask: "What's the PR URL?"

Parse `{owner}`, `{repo}`, and `{pr_number}` from the URL.

---

## Step 1: Fetch all comments

```bash
# Inline review thread comments
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
  --paginate \
  --jq '.[] | {id, path, line, body, user: .user.login, created_at, in_reply_to_id}'

# Review summaries
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
  --paginate \
  --jq '.[] | {id, state, body, user: .user.login, submitted_at}'

# Conversation tab comments
gh api repos/{owner}/{repo}/issues/{pr_number}/comments \
  --paginate \
  --jq '.[] | {id, body, user: .user.login, created_at}'

# PR metadata and current diff
gh pr view {pr_number} --json title,body,headRefName,baseRefName,files
gh pr diff {pr_number}
```

---

## Step 2: Triage — addressed vs unaddressed

For each comment thread, determine its status.

**Addressed** if any of:
- The thread is marked as resolved on GitHub
- The comment has a reply that acknowledges/closes it
- The diff shows the referenced code has since been changed in a way that responds to the concern

**Unaddressed** if:
- No reply or follow-up from the PR author
- The code at the referenced location is unchanged
- The reviewer re-raised the concern

Present a summary:
```
Found N comments:
  - X addressed / resolved
  - Y unaddressed (listed below)
```

---

## Step 3: Propose a response for each unaddressed comment

For each unaddressed comment, classify and propose:

```
[N] @reviewer — file.ts:42
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

## Step 4: Conversation

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

## Step 5: Execute

### Make code changes

Apply each approved code change. Match surrounding code conventions.

### Post replies

```bash
# Reply to an inline review comment
gh api repos/{owner}/{repo}/pulls/comments/{comment_id}/replies \
  -X POST \
  -f body="Your reply text"

# Reply as an issue comment (for conversation-tab comments)
gh api repos/{owner}/{repo}/issues/{pr_number}/comments \
  -X POST \
  -f body="Your reply text"
```

### Merge latest from base branch

```bash
git fetch origin {base_branch}
git merge origin/{base_branch}
```

If there are merge conflicts, resolve them and commit. Tell the user what was conflicted.

### Run all checks

Discover and run the full check suite (look in `AGENTS.md`, `package.json`, `Makefile`, etc.). Fix any failures.

### Push

Check for merged/closed PRs on this branch before pushing:
```bash
gh pr list --head {branch} --state merged --json number,title
```

Then push:
```bash
git push origin {branch}
```

Check PR state after pushing.

---

## Step 6: Report

```
Done.
  - Made N code changes
  - Posted N replies
  - Merged latest from {base_branch} (no conflicts / N conflicts resolved)
  - All checks passed
  - Pushed to origin/{branch}

PR: {pr_url}
```

---

## Notes

- Never post a reply without user approval
- Never push with failing checks
- The proposals are starting points — the user's preference always wins
