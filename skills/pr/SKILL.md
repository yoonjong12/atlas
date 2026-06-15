---
name: pr
description: "Access Bitbucket Cloud pull requests — view details, diffs, comments, reviews, post comments, and approve. This skill should be used when the user asks about a PR, wants to read PR comments or reviews, check a PR diff, post a comment, or approve a PR. Trigger on: 'PR', 'pull request', 'PR comments', 'review', 'PR 확인', 'PR 코멘트', '리뷰 확인', 'PR diff', '풀리퀘스트', 'PR에 코멘트 달아', 'approve PR', '승인'"
argument-hint: "<PR number or URL>"
allowed-tools: Bash, Read
---

# PR — Bitbucket Pull Request Workflow

Use this skill for Bitbucket Cloud PR lifecycle work: discover an existing PR, inspect its diff and comments, create a PR, write or revise its description, respond to review, approve, and merge.

Prefer `${CLAUDE_PLUGIN_ROOT}/scripts/bb_pr.sh` for supported operations. It auto-detects workspace/repo from `git remote get-url origin`; override with `WORKSPACE=... REPO_SLUG=...` when operating outside a checked-out repo.

## Prerequisites

`BITBUCKET_EMAIL` and `BITBUCKET_API_TOKEN` must be in env. If missing, run `/atlas:setup` before doing PR work.

## Operating Rules

- Inspect the current branch and dirty worktree before creating or merging PRs.
- If the user gives a PR URL, extract the number from `/pull-requests/<id>`.
- If the user omits a PR ID, run `bb_pr.sh find-by-branch` first.
- Use `@file` for long markdown descriptions, review replies, and merge messages.
- Summarize large diffs instead of pasting raw diff unless the user asks for exact output.
- Do not merge when there are unresolved review concerns, failing checks, or unclear target branch unless the user explicitly confirms.
- Follow `references/pr-conventions.md` for PR description structure and review reply format.

## Common Workflows

### Read or Review a PR

```bash
${CLAUDE_PLUGIN_ROOT}/scripts/bb_pr.sh get <pr_id>
${CLAUDE_PLUGIN_ROOT}/scripts/bb_pr.sh diff <pr_id>
${CLAUDE_PLUGIN_ROOT}/scripts/bb_pr.sh comments <pr_id>
${CLAUDE_PLUGIN_ROOT}/scripts/bb_pr.sh activity <pr_id>
```

Report findings first when reviewing code. Include file/line references from the diff when possible.

### Create a PR

Prepare the branch, push it, write a concise markdown body, then create:

```bash
${CLAUDE_PLUGIN_ROOT}/scripts/bb_pr.sh create "PR title" @/tmp/pr-body.md [source_branch] [destination_branch]
```

Defaults are `current_branch -> main`. The created PR closes the source branch on merge.

### Edit Title or Description

```bash
${CLAUDE_PLUGIN_ROOT}/scripts/bb_pr.sh update <pr_id> title "New title"
${CLAUDE_PLUGIN_ROOT}/scripts/bb_pr.sh update <pr_id> description @/tmp/pr-body.md
```

Use this when review feedback asks for a clearer PR description or when implementation scope changes.

### Comment or Reply to Review

```bash
${CLAUDE_PLUGIN_ROOT}/scripts/bb_pr.sh comment <pr_id> @/tmp/reply.md
${CLAUDE_PLUGIN_ROOT}/scripts/bb_pr.sh inline <pr_id> path/to/file.py 42 @/tmp/inline-reply.md
```

Before replying, read `comments` and `activity` so the response matches the actual review thread.

**`bb_pr.sh comment` posts a general comment — NOT a thread reply.**
To reply inside a reviewer's comment thread, use the Bitbucket API directly with `"parent": {"id": <comment_id>}`. See `references/bitbucket-api.md` → "Reply to Comment Thread" and `references/pr-conventions.md` → "Review Reply Convention" for the full workflow.

Steps to post a thread reply:
1. `bb_pr.sh comments <pr_id>` — find the parent comment ID (`.id` field of the reviewer's comment)
2. Write reply body to `/tmp/reply.md` following the `### Request N` format in `pr-conventions.md`
3. Post via raw API with `{"content": {"raw": "..."}, "parent": {"id": <id>}}`
4. Verify the reply appears nested under the reviewer's comment, not as a standalone comment

### Approve or Merge

```bash
${CLAUDE_PLUGIN_ROOT}/scripts/bb_pr.sh approve <pr_id>
${CLAUDE_PLUGIN_ROOT}/scripts/bb_pr.sh merge <pr_id> @/tmp/merge-message.md
```

Merge only after checking current PR state, review status, and any requested validation results.

## Subcommands

| Call | Purpose |
|------|---------|
| `bb_pr.sh get <pr_id>` | PR details, URL, and description |
| `bb_pr.sh diff <pr_id>` | unified diff |
| `bb_pr.sh comments <pr_id>` | general and inline comments grouped by file |
| `bb_pr.sh activity <pr_id>` | approvals, updates, and comment activity |
| `bb_pr.sh list [state]` | list PRs; default `OPEN` |
| `bb_pr.sh find-by-branch [branch]` | find open PR for branch; default current branch |
| `bb_pr.sh create <title> [desc\|@file] [source] [dest]` | create PR |
| `bb_pr.sh update <pr_id> title\|description <value\|@file>` | update PR metadata |
| `bb_pr.sh comment <pr_id> <body\|@file>` | post general comment |
| `bb_pr.sh inline <pr_id> <path> <line> <body\|@file>` | post inline comment |
| `bb_pr.sh approve <pr_id>` | approve PR |
| `bb_pr.sh merge <pr_id> [message\|@file]` | merge PR and close source branch |

## API Fallback

For edge cases not covered by the wrapper, call Bitbucket REST directly with Basic auth. Keep tokens out of logs and use `references/bitbucket-api.md` for endpoint shapes.
