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

The user provides a Jira ticket URL or key (e.g. `https://yourorg.atlassian.net/browse/PROJ-1234` or just `PROJ-1234`).

If no ticket is provided, ask: "What's the Jira ticket URL or key?"

---

## Step 1: Fetch the ticket

Use the Atlassian MCP tools to retrieve the ticket:
- Summary (title)
- Description
- Acceptance criteria (look in the description or a dedicated field)
- Any linked issues or parent epics for context

Extract the ticket key (e.g. `PROJ-1234`) and the Jira cloud ID if needed.

---

## Step 2: Understand the codebase

Before planning, explore the relevant parts of the codebase:
- Read `AGENTS.md` if it exists (project conventions, test commands, PR process)
- Read `.github/pull_request_template.md` or `.github/PULL_REQUEST_TEMPLATE.md` if it exists
- Explore directories and files relevant to the ticket's domain

---

## Step 3: Build and present an implementation plan

Produce a plan with:
1. **Summary**: What this ticket requires in plain terms
2. **Proposed commits**: A numbered list of logical, atomic commits. Each entry should say what changes and what tests cover it.
3. **Checks to run**: The lint, typecheck, and test commands for this project (inferred from `AGENTS.md`, `package.json`, `Makefile`, `mix.exs`, etc.)

Present the plan to the user. **Wait for approval before proceeding.** The user may edit the commit structure or scope before you begin.

---

## Step 4: Create the worktree and branch

Derive the branch name from the ticket:
- Format: `PROJ-1234/short-slug-from-summary` (lowercase, hyphen-separated, ≤5 words after the key)
- Example: `CARS-1234/add-inventory-filter`

```bash
git fetch origin main
git worktree add ../$(basename $(pwd))-PROJ-1234 -b PROJ-1234/short-slug origin/main
```

All subsequent implementation work happens inside the worktree directory.

If the repo has no worktree support or the user prefers a regular branch, fall back to `git checkout -b PROJ-1234/slug`.

---

## Step 5: Implement — commit by commit

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

## Step 6: Run all checks

Discover and run the project's full check suite. Look in this order:
1. `AGENTS.md` — explicit commands listed
2. `package.json` scripts (`lint`, `typecheck`, `test`)
3. `Makefile` targets
4. `mix.exs` (Elixir: `mix test`, `mix credo`, `mix dialyzer`)
5. Common defaults: `npm test`, `pytest`, `go test ./...`

Run each check. Fix any failures before proceeding. Do not push with failing checks.

---

## Step 7: Push and create a draft PR

### Check branch safety first

Before pushing:
```bash
gh pr list --head PROJ-1234/short-slug --state merged --json number,title
gh pr list --head PROJ-1234/short-slug --state closed --json number,title
```

If a merged/closed PR exists on this branch, create a new branch and push from there instead.

Rebase onto latest base branch before pushing:
```bash
git fetch origin main
git rebase origin/main
git push -u origin PROJ-1234/short-slug
```

Check PR state again after pushing.

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
  --base main \
  --draft \
  --title "type: short description [PROJ-1234]" \
  --body "..."
```

Report the PR URL to the user.

---

## Step 8: Advance the ticket status

After the draft PR is created, move the Jira ticket to the appropriate in-progress state.

### Detect available tooling (in order of preference)

**1. Atlassian MCP** (preferred — already used in Step 1):
```
# Get available transitions
getTransitionsForJiraIssue(cloudId, issueIdOrKey: "PROJ-1234")

# Apply the transition
transitionJiraIssue(cloudId, issueIdOrKey: "PROJ-1234", transition: { id: "<transition_id>" })
```

**2. `acli` (Atlassian CLI)**:
```bash
which acli && acli jira issue transition "PROJ-1234" --transition "In Progress"
```

**3. `jira` CLI**:
```bash
which jira && jira issue move "PROJ-1234" "In Progress"
```

**4. `curl` against the Jira REST API** (fallback if no tooling is available):
```bash
# Get transitions
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "https://yourorg.atlassian.net/rest/api/3/issue/PROJ-1234/transitions" | jq '.transitions[] | {id, name}'

# Apply transition
curl -s -X POST \
  -u "$JIRA_USER:$JIRA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"transition": {"id": "<transition_id>"}}' \
  "https://yourorg.atlassian.net/rest/api/3/issue/PROJ-1234/transitions"
```

### Choosing the right transition

Fetch the available transitions first and pick the one that best matches "In Progress" (common names: `In Progress`, `In Development`, `Start Progress`, `Start`). Do not guess a transition ID.

If none of the tooling is available and `JIRA_USER`/`JIRA_TOKEN` env vars are not set, skip this step and tell the user: "Could not advance ticket status — no Jira tooling available. You can move it manually."

If the transition succeeds, confirm: "Moved PROJ-1234 → In Progress."

---

## Notes

- Never merge — always leave the PR as a draft for human review
- If the ticket has no clear acceptance criteria, ask the user to clarify scope before planning
- The plan step is a checkpoint — don't skip it even if the ticket seems simple
