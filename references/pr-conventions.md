# PR Conventions

Conventions for PR descriptions and review feedback responses on this project.

---

## PR Description Structure

Sections in order:

| Section | Purpose |
|---|---|
| **Summary** | 1–2 sentences. What this PR does and why in one breath. |
| **Why** | Problem being solved. What was broken or missing before. |
| **ASIS → TOBE** | Key diffs, visualized appropriately (see below). |
| **Changes** | Table of files changed + what changed. Caveman compact. |
| **Benchmark** | Perf or cost impact. If structural, verify via test mock instead of re-run. |
| **Validation** | pytest count, lint, branch state. |

### Title format

```
WAO-XXX LYZ: short description
```

- `LYZ` = sprint + subtask index (e.g. L3, L3Z = story + subtask)
- Keep title under ~60 chars

### Writing style

- Sentences: concise, naturally flowing. Not choppy short fragments strung together.
- Changes section: table format, caveman compact (drop articles, short synonyms).
- No internal ticket codes, section symbols (§), or jargon in output visible to reviewers.

### ASIS → TOBE visualization

Match the visualization to what actually changed:

| Change type | Visualization |
|---|---|
| Architecture / backend logic | Code diff block (before/after) |
| Frontend / UI use-case | Screenshot pair |
| Backend API contract | API request/response diff |

**Screenshot rules:**
- Always label the condition: what env vars were set, what file was added/removed.
- Screenshot alone is not enough — explain what's different and why it matters.
- Example label: `**Before** — THELOOP_JUDGE_RUBRICS_DIR unset; only built-in axes appear`

---

## Review Reply Convention

### Step-by-step workflow

1. **Read first** — `bb_pr.sh comments <pr_id>` + `bb_pr.sh activity <pr_id>`
2. **Find parent comment ID** — the `.id` of the reviewer's comment you're replying to
3. **Categorize each request** — Fixed / Deferred / Disagreed (with reason)
4. **Write reply** to `/tmp/reply.md` following the format below
5. **Post as thread reply** via raw Bitbucket API with `"parent": {"id": <id>}` (NOT `bb_pr.sh comment`)
6. **Verify** the reply appears nested under the reviewer's thread

> `bb_pr.sh comment` posts a standalone general comment, not a thread reply.
> Always use the raw API when replying to a reviewer's comment.

### Reply format

```markdown
### Request 1: <short title> — Fixed

**Before**
```python
# old code
```

**After**
```python
# new code
```

One sentence explaining what changed and why.

---

### Request 2: <short title> — Deferred

One sentence on why this is deferred and what ticket/PR will address it.
```

Rules:
- `### Request N: Title — Fixed` or `— Deferred` or `— Disagree (reason)`
- Before/After code blocks for Fixed items where code changed
- One sentence per request — no multi-paragraph explanations
- Writing style: caveman (concise, no filler), sentences ≤ ~10 words each
- If multiple requests from one reviewer, batch into a single reply

### Posting the reply (raw API)

```bash
# Write payload to file to avoid shell escaping issues
python3 -c "
import json
body = open('/tmp/reply.md').read()
print(json.dumps({'content': {'raw': body}, 'parent': {'id': PARENT_ID}}))
" > /tmp/reply-payload.json

curl -s -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  -X POST \
  -H "Content-Type: application/json" \
  -d @/tmp/reply-payload.json \
  "https://api.bitbucket.org/2.0/repositories/${WORKSPACE}/${REPO_SLUG}/pullrequests/${PR_ID}/comments" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('reply id:', d['id'], '| parent:', d.get('parent',{}).get('id'))"
```

If you accidentally post as a general comment:
1. Note the comment ID from the response
2. Delete it: `curl -X DELETE ... /comments/<id>`
3. Re-post with `"parent"` field

See `references/bitbucket-api.md` → "Reply to Comment Thread" and "Delete PR Comment" for full curl commands.
