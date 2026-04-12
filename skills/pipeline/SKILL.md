---
name: pipeline
description: "Check Bitbucket Cloud pipeline status, wait for completion, and diagnose failures. This skill should be used when the user asks about CI/CD status, pipeline results, build failures, or after pushing code. Trigger on: 'pipeline', 'pipeline status', 'CI', 'build status', '파이프라인', '파이프라인 확인', 'check pipeline', 'build failed', '빌드 확인', 'CI 결과'"
argument-hint: ""
allowed-tools: Bash, Read
---

# Pipeline — Bitbucket CI Status

Check Bitbucket Cloud pipeline status, monitor running builds, and diagnose failures.

## Prerequisites

Requires `BITBUCKET_EMAIL` and `BITBUCKET_API_TOKEN` environment variables. If not set, direct the user to run `/atlassian-extended:setup`.

## Process

### Step 1: Detect Repository

Extract workspace and repo slug from git remote:

```bash
REMOTE_URL=$(git remote get-url origin)
WORKSPACE=$(echo "$REMOTE_URL" | sed -E 's#.*bitbucket.org[:/]([^/]+)/.*#\1#')
REPO_SLUG=$(echo "$REMOTE_URL" | sed -E 's#.*bitbucket.org[:/][^/]+/([^.]+).*#\1#')
echo "Workspace: $WORKSPACE | Repo: $REPO_SLUG"
```

### Step 2: Get Latest Pipeline

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pipelines/?sort=-created_on&pagelen=1"
```

Extract from response:
- `.values[0].uuid` — pipeline UUID
- `.values[0].state.name` — `PENDING`, `RUNNING`, or `COMPLETED`
- `.values[0].state.result.name` — `SUCCESSFUL`, `FAILED`, `STOPPED` (when completed)
- `.values[0].target.commit.hash` — verify it matches the pushed commit

If the commit hash does not match the latest local commit (`git rev-parse HEAD`), inform the user that the pipeline is for a different commit.

### Step 3: Monitor (if running)

If state is `PENDING` or `RUNNING`, poll in the background:

```bash
# Run in background with 60-90 second intervals
while true; do
  STATE=$(curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
    "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pipelines/${PIPELINE_UUID}" \
    | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['state']['name'])")
  if [ "$STATE" = "COMPLETED" ]; then break; fi
  sleep 75
done
```

Use `run_in_background: true` for the Bash tool to avoid blocking.

### Step 4: Get Step Details

```bash
curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pipelines/${PIPELINE_UUID}/steps/"
```

Present a status table:

```
| Step              | Status     | Duration |
|-------------------|------------|----------|
| Build             | SUCCESSFUL | 45s      |
| Test              | FAILED     | 120s     |
| Deploy            | NOT_RUN    | -        |
```

### Step 5: Diagnose Failures

For each failed step, fetch logs:

```bash
curl -s -L -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  -H "Range: bytes=0-999999" \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pipelines/${PIPELINE_UUID}/steps/${STEP_UUID}/log" \
  | strings | grep -i -E "(error|fail|exception|traceback)" | tail -50
```

**Notes on log fetching:**
- Log endpoint returns binary data — pipe through `strings` to extract readable text
- Use `-L` flag to follow 307 redirects
- Use `Range` header to limit download size
- Extract error-relevant lines with grep

Present the error lines and propose a diagnosis.

## Additional Resources

- **`references/bitbucket-api.md`** — Full Bitbucket REST API endpoint reference
