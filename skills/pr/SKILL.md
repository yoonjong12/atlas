---
name: pr
description: "Access Bitbucket Cloud pull requests — view details, diffs, comments, reviews, post comments, and approve. This skill should be used when the user asks about a PR, wants to read PR comments or reviews, check a PR diff, post a comment, or approve a PR. Trigger on: 'PR', 'pull request', 'PR comments', 'review', 'PR 확인', 'PR 코멘트', '리뷰 확인', 'PR diff', '풀리퀘스트', 'PR에 코멘트 달아', 'approve PR', '승인'"
argument-hint: "<PR number or URL>"
allowed-tools: Bash, Read
---

# PR — Bitbucket Pull Request Access

View PR details, diffs, comments, and reviews. Post comments on PRs.

## Prerequisites

Requires `BITBUCKET_EMAIL` and `BITBUCKET_API_TOKEN` environment variables. If not set, direct the user to run `/atlassian-extended:setup`.

## Repo Detection

Extract workspace and repo slug from git remote:

```bash
REMOTE_URL=$(git remote get-url origin)
WORKSPACE=$(echo "$REMOTE_URL" | sed -E 's#.*bitbucket.org[:/]([^/]+)/.*#\1#')
REPO_SLUG=$(echo "$REMOTE_URL" | sed -E 's#.*bitbucket.org[:/][^/]+/([^.]+).*#\1#')
```

## Read Operations

### PR Details

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pullrequests/${PR_ID}"
```

Present: title, author, source → destination branch, state, created/updated dates.

### PR Diff

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pullrequests/${PR_ID}/diff"
```

Returns unified diff format. Summarize changed files and key modifications.

### PR Comments

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pullrequests/${PR_ID}/comments?pagelen=100"
```

For each comment, extract `.content.raw`, `.user.display_name`, `.created_on`.
For inline comments, also extract `.inline.path`, `.inline.from`, `.inline.to`.

Present comments grouped by:
1. General comments (no `.inline` field)
2. Inline comments (grouped by file path)

### PR Activity / Reviews

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pullrequests/${PR_ID}/activity?pagelen=50"
```

Activity types in response:
- `.approval` — who approved and when
- `.update` — state changes (open, merged, declined)
- `.comment` — comment activity

### List PRs

```bash
# Open PRs
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pullrequests?state=OPEN&pagelen=10"
```

States: `OPEN`, `MERGED`, `DECLINED`, `SUPERSEDED`

## Write Operations

### Post General Comment

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  -X POST -H "Content-Type: application/json" \
  -d '{"content": {"raw": "Comment text in markdown"}}' \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pullrequests/${PR_ID}/comments"
```

### Post Inline Comment

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  -X POST -H "Content-Type: application/json" \
  -d '{"content": {"raw": "Comment on this line"}, "inline": {"path": "src/main.py", "to": 42}}' \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pullrequests/${PR_ID}/comments"
```

### Update PR Description

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  -X PUT -H "Content-Type: application/json" \
  -d '{"description": "Updated description in markdown"}' \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pullrequests/${PR_ID}"
```

Can also update title: `{"title": "New title", "description": "New description"}`

### Approve PR

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  -X POST \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pullrequests/${PR_ID}/approve"
```

## PR Number Detection

If the user provides a Bitbucket URL instead of a PR number, extract the PR ID:

```
https://bitbucket.org/workspace/repo/pull-requests/123 → PR_ID=123
```

If no PR number is given, check the current branch:

```bash
BRANCH=$(git branch --show-current)
# Search for open PRs from this branch
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pullrequests?state=OPEN" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
branch = '${BRANCH}'
for pr in data.get('values', []):
    if pr['source']['branch']['name'] == branch:
        print(f\"PR #{pr['id']}: {pr['title']}\")
"
```

## Additional Resources

- **`references/bitbucket-api.md`** — Full Bitbucket REST API endpoint reference
