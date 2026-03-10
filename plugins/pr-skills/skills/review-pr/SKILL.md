---
name: review-pr
description: >-
  Given a PR link, evaluates it as a code reviewer. Surfaces concerns, questions,
  and suggestions with file path, line number, a summary, and a copy-paste-ready
  GitHub comment. Conversational — the user can ask questions about any finding.
  Never posts comments; read-only output for human review.
disable-model-invocation: true
---

# Review PR

## Input

The user provides a GitHub PR URL as an argument or in conversation (e.g. `/review-pr https://github.com/org/repo/pull/123`).

If `$ARGUMENTS` is provided, treat it as the PR URL.

If nothing is provided, ask: "What's the PR URL?"

Parse `{owner}`, `{repo}`, and `{pr_number}` from the URL.

---

## Prerequisites

Before starting, verify:
- `gh auth status` — must be authenticated to GitHub

If `gh` is not authenticated, stop and tell the user: "Run `gh auth login` first."

---

## Step 1: Fetch PR context

Use FetchUrl with the PR URL as a first option — Droid has native GitHub PR integration that returns structured metadata and diff:

```
FetchUrl: https://github.com/{owner}/{repo}/pull/{pr_number}
```

Supplement with `gh` CLI for data FetchUrl doesn't provide:

```bash
# CI check status
gh pr checks {pr_number} --repo {owner}/{repo}

# Existing review comments (to avoid duplicating what's already been said)
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
  --paginate \
  --jq '.[] | {path, line, body, user: .user.login}'

gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
  --paginate \
  --jq '.[] | {state, body, user: .user.login}'
```

Also read `AGENTS.md` if present in the repo — it may describe project conventions relevant to the review.

---

## Step 2: Note CI status

Before reviewing code, surface the current CI state:

```bash
gh pr checks {pr_number} --repo {owner}/{repo}
```

Classify:
- **All passing** — no CI concerns to note
- **Some failing** — list the failing check names; include them in the findings list as `blocking` items (e.g. "[N] CI — tests/lint-check — blocking: CI is currently failing")
- **Pending** — note that CI is still running; review findings may be incomplete

---

## Step 3: Review the diff

Analyze the diff file-by-file. For each changed file, evaluate:

- **Correctness**: Logic bugs, off-by-one errors, wrong conditions, incorrect assumptions
- **Error handling**: Missing null checks, unhandled promise rejections, uncaught exceptions, missing 404/error responses
- **Test coverage**: New code paths without tests, edge cases not covered, tests that only test happy paths
- **Security**: SQL injection, XSS, IDOR, insecure direct object references, exposed secrets, improper auth checks
- **Performance**: N+1 queries, unnecessary re-renders, missing indexes, large payloads
- **Conventions**: Style inconsistencies with the surrounding codebase, naming that doesn't match project patterns
- **Public API docs**: Exported functions, components, or endpoints missing documentation

Do not comment on things that are already flagged by existing review comments.

If the PR is large (many files), prioritize the highest-risk areas (auth, data mutations, new APIs) and note that lower-risk files were skipped.

---

## Step 4: Present findings

Present all findings as a numbered list. Lead with CI status if relevant, then code findings.

Each finding must include:

```
[N] path/to/file.ts:LINE — SEVERITY
Summary: one-line description of the concern
Proposed comment:
> The comment text, written as if you're the reviewer posting it on GitHub.
> Specific, constructive, and actionable. Reference the relevant code by
> quoting or describing it. Suggest a fix when possible.
```

**Severity levels:**
- `blocking` — must be addressed before merge (bug, security issue, missing critical test, failing CI)
- `suggestion` — improvement worth making but not a blocker
- `question` — genuinely unclear; reviewer needs more context before deciding

**Formatting rules for proposed comments:**
- Write in second person ("Consider...", "This will throw if...", "Why is this...")
- Be specific — reference the exact variable, function, or pattern
- Suggest a concrete fix when you have one
- Keep it under 4 sentences for suggestions; longer is fine for complex bugs

If the PR looks good with no meaningful concerns, say so clearly rather than inventing minor nits.

---

## Step 5: Conversation

After presenting findings, remain available for follow-up:

- The user can ask "tell me more about #3" — expand with more detail, context, or alternatives
- The user can ask "is #5 really blocking?" — re-evaluate and give your honest assessment
- The user can ask "draft a revised comment for #2 that's less harsh" — rewrite it
- The user can ask general questions about the PR ("what does this PR actually do?")

Stay in this conversational mode until the user is done.

---

## Notes

- This skill is read-only. It never posts comments to GitHub, modifies code, or creates reviews.
- The user decides what to post and when. The proposed comments are copy-paste starting points.
- Do not pad the findings list. Five real concerns are more useful than ten manufactured ones.
