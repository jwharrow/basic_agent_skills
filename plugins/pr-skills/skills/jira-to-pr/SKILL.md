---
name: jira-to-pr
description: >-
  Full workflow from a Jira ticket to a draft PR. Takes a Jira link, plans the
  implementation, creates a git worktree on the Jira-suggested branch, makes
  logical commits with tests, runs all checks, pushes, and opens a draft PR.
  Use when the user wants to start work on a Jira ticket end-to-end.
disable-model-invocation: true
---

# Jira to Draft PR

## Input

The user provides a Jira ticket URL or key as an argument or in conversation (e.g. `/jira-to-pr PROJ-1234` or `https://yourorg.atlassian.net/browse/PROJ-1234`).

If `$ARGUMENTS` is provided, treat it as the ticket URL or key.

If nothing is provided, ask: "What's the Jira ticket URL or key?"

---

## Prerequisites

Before starting, verify:
- `gh auth status` — must be authenticated to GitHub
- Atlassian MCP tools must be available (used to fetch the ticket and advance its status)
- Must be run from inside a git repository

If `gh` is not authenticated, stop and tell the user: "Run `gh auth login` first."
If the Atlassian MCP is unavailable, you can still proceed but skip Step 8 (status transition).

---

## Step 1: Fetch the ticket

Use the Atlassian MCP tools to retrieve the ticket:
- Summary (title)
- Description
- Acceptance criteria (look in the description or a dedicated field)
- Any linked issues or parent epics for context
- The current assignee

Extract the ticket key (e.g. `PROJ-1234`) and the Jira cloud ID if needed.

---

## Step 2: Detect the default branch

```bash
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef -q '.defaultBranchRef.name')
```

Use `$DEFAULT_BRANCH` everywhere `main` appears in subsequent steps.

---

## Step 3: Understand the codebase

Before planning, explore the relevant parts of the codebase:
- Read `AGENTS.md` if it exists (project conventions, test commands, PR process)
- Read `.github/pull_request_template.md` or `.github/PULL_REQUEST_TEMPLATE.md` if it exists
- Explore directories and files relevant to the ticket's domain

---

## Step 4: Build and present an implementation plan

Produce a plan with:
1. **Summary**: What this ticket requires in plain terms
2. **Proposed commits**: A numbered list of logical, atomic commits. Each entry should say what changes and what tests cover it.
3. **Checks to run**: The lint, typecheck, and test commands for this project (inferred from `AGENTS.md`, `package.json`, `Makefile`, `mix.exs`, etc.)

Present the plan to the user. **Wait for approval before proceeding.** The user may edit the commit structure or scope before you begin.

---

## Step 5: Create the worktree and branch

Derive the branch name from the ticket:
- Format: `PROJ-1234/short-slug-from-summary` (lowercase, hyphen-separated, ≤5 words after the key)
- Example: `CARS-1234/add-inventory-filter`

Use a deterministic worktree location under `.worktrees/` in the repo root:

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
WORKTREE_PATH="$REPO_ROOT/.worktrees/PROJ-1234"

git fetch origin "$DEFAULT_BRANCH"
git worktree add "$WORKTREE_PATH" -b PROJ-1234/short-slug "origin/$DEFAULT_BRANCH"
```

All subsequent implementation work happens inside `$WORKTREE_PATH`.

**Cleanup note**: After the PR merges, remove the worktree with:
```bash
git worktree remove "$REPO_ROOT/.worktrees/PROJ-1234"
git branch -d PROJ-1234/short-slug
```

If worktrees are not supported or the user prefers a regular branch, fall back to:
```bash
git fetch origin "$DEFAULT_BRANCH"
git checkout -b PROJ-1234/short-slug "origin/$DEFAULT_BRANCH"
```

---

## Step 6: Implement — commit by commit

Work through the approved commit plan:

- Make changes for commit N
- Write or update tests for commit N
- Stage and commit:
  ```bash
  git add -A
  git commit -m "type(scope): description [PROJ-1234]"
  ```
- Repeat for each planned commit

Commit message format: conventional commits (`feat`, `fix`, `refactor`, `chore`, `test`, `docs`, `ci`) with the Jira key in brackets at the end.

Do not make a single monolithic commit. Each commit should be meaningful and self-contained.

---

## Step 7: Run all checks

Discover and run the project's full check suite. Look in this order:
1. `AGENTS.md` — explicit commands listed
2. `package.json` scripts (`lint`, `typecheck`, `test`)
3. `Makefile` targets
4. `mix.exs` (Elixir: `mix test`, `mix credo`, `mix dialyzer`)
5. Common defaults: `npm test`, `pytest`, `go test ./...`

Run each check. Fix any failures before proceeding. Do not push with failing checks.

---

## Step 8: Push and create a draft PR

### Branch safety checks

Before pushing, verify there is no existing merged or closed PR on this branch:

```bash
BRANCH="PROJ-1234/short-slug"
gh pr list --head "$BRANCH" --state merged --json number,title
gh pr list --head "$BRANCH" --state closed --json number,title
```

If a merged/closed PR exists on this branch, create a new branch from HEAD and push from there instead.

Rebase onto the latest default branch before pushing:

```bash
git fetch origin "$DEFAULT_BRANCH"
git rebase "origin/$DEFAULT_BRANCH"
```

Verify only the intended commits are ahead:

```bash
git log --oneline "origin/$DEFAULT_BRANCH..HEAD"
```

Push:

```bash
git push -u origin "$BRANCH"
```

After pushing, re-check PR state — a PR can be merged between the pre-push check and the actual push:

```bash
gh pr list --head "$BRANCH" --state merged --json number,title
```

Never use `git push --force`. Always use `git push --force-with-lease` if a force push is needed after rebasing.

### Write the PR description

If a PR template exists (`.github/pull_request_template.md`), fill every section with real content. Never leave placeholder text.

If no template, use:
```markdown
## Summary

[What this PR does and why. Reference the Jira ticket naturally.]

## Changes

[Grouped by logical area. Be specific — name files/modules when it helps the reviewer navigate.]

## Testing

[How changes were verified: test commands run, results, manual steps.]

## Jira

https://yourorg.atlassian.net/browse/PROJ-1234
```

### Create the draft PR

```bash
gh pr create \
  --base "$DEFAULT_BRANCH" \
  --draft \
  --title "type: short description [PROJ-1234]" \
  --body "..."
```

Report the PR URL to the user.

---

## Step 9: Advance the ticket status and assign

After the draft PR is created, update the Jira ticket.

### Assign the ticket to yourself

Use the Atlassian MCP to get your account ID and assign:
```
editJiraIssue(cloudId, issueIdOrKey: "PROJ-1234", fields: { assignee: { accountId: "<your_account_id>" } })
```

To get your account ID: `atlassianUserInfo()`

### Advance the ticket status

Use the Atlassian MCP (preferred):
```
# Get available transitions
getTransitionsForJiraIssue(cloudId, issueIdOrKey: "PROJ-1234")

# Apply the best match for "In Progress"
transitionJiraIssue(cloudId, issueIdOrKey: "PROJ-1234", transition: { id: "<transition_id>" })
```

If the Atlassian MCP is unavailable, try in order:

**`acli`:**
```bash
which acli && acli jira issue transition "PROJ-1234" --transition "In Progress"
```

**`jira` CLI:**
```bash
which jira && jira issue move "PROJ-1234" "In Progress"
```

**`curl` fallback:**
```bash
# Get transitions
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "https://yourorg.atlassian.net/rest/api/3/issue/PROJ-1234/transitions" \
  | jq '.transitions[] | {id, name}'

# Apply transition
curl -s -X POST \
  -u "$JIRA_USER:$JIRA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"transition": {"id": "<transition_id>"}}' \
  "https://yourorg.atlassian.net/rest/api/3/issue/PROJ-1234/transitions"
```

Always fetch available transitions first — never guess a transition ID. Pick the best match for "In Progress" (common names: `In Progress`, `In Development`, `Start Progress`, `Start`).

If no tooling is available and `JIRA_USER`/`JIRA_TOKEN` are not set, skip and tell the user: "Could not update ticket — no Jira tooling available. Move it manually."

If successful, confirm: "Assigned PROJ-1234 to you and moved → In Progress."

---

## Notes

- Never merge — always leave the PR as a draft for human review
- If the ticket has no clear acceptance criteria, ask the user to clarify scope before planning
- The plan step (Step 4) is a checkpoint — don't skip it even if the ticket seems simple
