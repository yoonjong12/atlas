---
name: repo
description: "Bitbucket repository operations — clone, info, branches, commits, file contents. This skill should be used when the user asks to 'clone a repo', 'list branches', 'show commits', 'view repo info', 'get file from bitbucket', or references a Bitbucket repository. Trigger on: 'repo', 'clone', 'branch', 'branches', '브랜치', '레포', '클론', 'repo info', 'commits', 'bitbucket repo', '레포 정보'"
argument-hint: "<repo slug, branch name, or Bitbucket URL>"
allowed-tools: Bash, Read
---

# Repo — Bitbucket Repository Operations

Clone, inspect branches, view commits, and read file contents from Bitbucket Cloud repositories.

## Prerequisites

Requires `BITBUCKET_EMAIL` and `BITBUCKET_API_TOKEN` environment variables. If not set, direct the user to run `/atlas:setup`.

## Repo Detection

Extract workspace and repo slug from git remote or user input:

```bash
REMOTE_URL=$(git remote get-url origin)
WORKSPACE=$(echo "$REMOTE_URL" | sed -E 's#.*bitbucket.org[:/]([^/]+)/.*#\1#')
REPO_SLUG=$(echo "$REMOTE_URL" | sed -E 's#.*bitbucket.org[:/][^/]+/([^.]+).*#\1#')
echo "Workspace: $WORKSPACE | Repo: $REPO_SLUG"
```

If the user provides a Bitbucket URL, extract from it:
```
https://bitbucket.org/mindai/pcr_skill_networking → WORKSPACE=mindai, REPO_SLUG=pcr_skill_networking
```

## Read Operations

### Repo Info

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'Name: {d[\"name\"]}')
print(f'Full name: {d[\"full_name\"]}')
print(f'Language: {d.get(\"language\", \"N/A\")}')
print(f'Main branch: {d.get(\"mainbranch\", {}).get(\"name\", \"N/A\")}')
print(f'Updated: {d[\"updated_on\"]}')
print(f'Size: {d.get(\"size\", \"N/A\")} bytes')
"
```

### List Branches

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/refs/branches?pagelen=25&sort=-target.date" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
for b in d.get('values', []):
    date = b['target']['date'][:10]
    print(f'{b[\"name\"]:40s} {date}  {b[\"target\"][\"hash\"][:8]}')
"
```

### List Commits

```bash
# Recent commits on a branch (default: main)
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/commits/${BRANCH}?pagelen=10" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
for c in d.get('values', []):
    date = c['date'][:10]
    msg = c['message'].split('\n')[0][:60]
    author = c['author'].get('user', {}).get('display_name', c['author'].get('raw', 'unknown'))
    print(f'{c[\"hash\"][:8]}  {date}  {author:20s}  {msg}')
"
```

### View File Contents

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/src/${BRANCH}/${FILE_PATH}"
```

### List Directory Contents

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/src/${BRANCH}/${DIR_PATH}?pagelen=100" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
for entry in d.get('values', []):
    kind = 'D' if entry['type'] == 'commit_directory' else 'F'
    print(f'{kind}  {entry[\"path\"]}')
"
```

### Compare Commits / Diff

```bash
# Diff between two refs (branches, tags, commits)
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/diff/${SOURCE}..${DESTINATION}"
```

## Write Operations

### Clone Repository

Three auth methods — pick based on what's configured:

**(a) SSH** (preferred when SSH key registered with Bitbucket):
```bash
git clone git@bitbucket.org:${WORKSPACE}/${REPO_SLUG}.git
```

**(b) HTTPS with Atlassian API token** (ATATT format — what `atlas:setup` issues):
```bash
# NOTE: ATATT tokens require `x-bitbucket-api-token-auth` as the username —
# NOT `x-token-auth` (that's for Bitbucket's older Repository Access Tokens).
git clone "https://x-bitbucket-api-token-auth:${BITBUCKET_API_TOKEN}@bitbucket.org/${WORKSPACE}/${REPO_SLUG}.git"
```

**(c) HTTPS with legacy app password** (older creds, still works):
```bash
git clone "https://${BITBUCKET_EMAIL}:${APP_PASSWORD}@bitbucket.org/${WORKSPACE}/${REPO_SLUG}.git"
```

**Specific branch**: add `-b <branch>` before the URL, e.g. `git clone -b release/internal <url>`.

**Common failure**: `fatal: Authentication failed` when using `x-token-auth:<ATATT-token>` → wrong user prefix. Switch to `x-bitbucket-api-token-auth`. After a successful HTTPS clone with token-in-URL, the token is persisted in `.git/config` — rewrite the remote (`git remote set-url origin https://bitbucket.org/${WORKSPACE}/${REPO_SLUG}.git`) and rely on a credential helper if you don't want the token on disk.

### Create Branch (via API)

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  -X POST -H "Content-Type: application/json" \
  -d "{\"name\": \"${NEW_BRANCH}\", \"target\": {\"hash\": \"${FROM_COMMIT_OR_BRANCH}\"}}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/refs/branches"
```

### Delete Branch (via API)

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  -X DELETE \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/refs/branches/${BRANCH_NAME}"
```

## List Repositories in Workspace

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}?pagelen=25&sort=-updated_on" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
for r in d.get('values', []):
    print(f'{r[\"slug\"]:30s} {r.get(\"language\", \"\"):10s} {r[\"updated_on\"][:10]}')
"
```

## Additional Resources

- **`references/bitbucket-api.md`** — Full Bitbucket REST API endpoint reference
