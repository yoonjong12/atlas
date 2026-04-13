---
name: setup
description: "This skill should be used when the user needs to configure Atlassian credentials, connect to Bitbucket or Jira for the first time, or troubleshoot authentication issues. Trigger on: 'setup', 'configure atlassian', 'set up bitbucket', 'set up jira', 'atlassian 설정', '빗버킷 설정', '지라 설정', 'connect atlassian', 'credentials not working', 'authentication failed', '인증 실패'"
argument-hint: ""
allowed-tools: Bash, Read, ToolSearch, AskUserQuestion
---

# Setup — Atlassian Integration Configuration

Configure credentials for Bitbucket REST API and Jira MCP access, then verify connectivity.

## Prerequisites

Two credential sets are required:

| Service | Credential | Location | Purpose |
|---------|-----------|----------|---------|
| Jira MCP | `JIRA_USERNAME` + `JIRA_API_TOKEN` | `~/.claude.json` → `mcpServers.atlassian` | Issue operations via mcp-atlassian |
| Bitbucket REST API | `BITBUCKET_EMAIL` + `BITBUCKET_API_TOKEN` | Shell environment variables | Pipeline, PR operations via curl |

## Process

### Step 1: Check Jira MCP

Verify `~/.claude.json` has the atlassian MCP server configured:

```bash
python3 -c "
import json
with open('$HOME/.claude.json') as f:
    d = json.load(f)
mcp = d.get('mcpServers', {}).get('atlassian', {})
if mcp:
    env = mcp.get('env', {})
    print(f'JIRA_URL: {env.get(\"JIRA_URL\", \"(not set)\")}')
    print(f'JIRA_USERNAME: {env.get(\"JIRA_USERNAME\", \"(not set)\")}')
    print(f'JIRA_API_TOKEN: {\"(set)\" if env.get(\"JIRA_API_TOKEN\") else \"(not set)\"}')
else:
    print('atlassian MCP server not configured')
"
```

If missing, guide the user:

1. Go to https://id.atlassian.com/manage-profile/security/api-tokens
2. Create an API token
3. Agent writes the config to `~/.claude.json`:

```bash
python3 -c "
import json
with open('$HOME/.claude.json') as f:
    d = json.load(f)
d.setdefault('mcpServers', {})['atlassian'] = {
    'command': 'uvx',
    'args': ['mcp-atlassian'],
    'env': {
        'JIRA_URL': 'https://mindai.atlassian.net',
        'JIRA_USERNAME': '<email>',
        'JIRA_API_TOKEN': '<token>',
        'CONFLUENCE_URL': 'https://mindai.atlassian.net/wiki',
        'CONFLUENCE_USERNAME': '<email>',
        'CONFLUENCE_API_TOKEN': '<token>'
    }
}
with open('$HOME/.claude.json', 'w') as f:
    json.dump(d, f, indent=2)
"
```

4. Restart Claude Code (`/exit` then relaunch)

### Step 2: Verify Jira Connectivity

After restart, test MCP connection:

```
ToolSearch({ query: "+atlassian jira_search" })
```

Then run a test query:

```typescript
jira_search({
  jql: "project = WAO AND type = Epic ORDER BY updated DESC",
  limit: 1,
  fields: "summary,status"
})
```

### Step 3: Check Bitbucket Credentials

Verify environment variables are set:

```bash
echo "BITBUCKET_EMAIL: ${BITBUCKET_EMAIL:-(not set)}"
echo "BITBUCKET_API_TOKEN: ${BITBUCKET_API_TOKEN:+(set)}"
```

If missing, guide the user:

1. Go to https://bitbucket.org/account/settings/app-passwords/
2. Create an app password with scopes:
   - `repository:read`
   - `pullrequest:read`
   - `pullrequest:write`
   - `pipeline:read`
3. Set environment variables in shell profile:
   ```bash
   export BITBUCKET_EMAIL="user@example.com"
   export BITBUCKET_API_TOKEN="ATBBxxxxxxxx"
   ```

### Step 4: Verify Bitbucket Connectivity

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/user"
```

- `200` → Connected
- `401` → Invalid credentials
- `403` → Missing scopes

### Step 5: Report Status

Present a summary table:

```
| Service    | Status      | Account           |
|------------|-------------|-------------------|
| Jira MCP   | Connected   | user@example.com  |
| Bitbucket  | Connected   | user@example.com  |
```

If any service is not connected, list the specific steps needed to fix it.
