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

| Service | Credential | Purpose |
|---------|-----------|---------|
| Bitbucket REST API | `BITBUCKET_EMAIL` + `BITBUCKET_API_TOKEN` | Pipeline, PR operations |
| Jira MCP | Atlassian OAuth (via MCP server) | Issue operations |

## Process

### Step 1: Check Bitbucket Credentials

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

### Step 2: Verify Bitbucket Connectivity

Test API access:

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -u "${BITBUCKET_EMAIL}:${BITBUCKET_API_TOKEN}" \
  "https://api.bitbucket.org/2.0/user"
```

- `200` → Connected
- `401` → Invalid credentials
- `403` → Missing scopes

### Step 3: Verify Jira MCP

Load and test the Atlassian MCP authenticate tool:

```
ToolSearch({ query: "+atlassian authenticate" })
```

Run the authenticate tool if not yet authenticated. Then verify with a test query:

```
ToolSearch({ query: "+atlassian atlassianUserInfo" })
atlassianUserInfo({})
```

If authentication fails, the user must complete the OAuth flow in their browser when prompted by the MCP server.

### Step 4: Report Status

Present a summary table:

```
| Service    | Status      | Account           |
|------------|-------------|-------------------|
| Bitbucket  | Connected   | user@example.com  |
| Jira MCP   | Connected   | User Name         |
```

If any service is not connected, list the specific steps needed to fix it.
