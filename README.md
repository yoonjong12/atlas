# atlassian-extended

Unified Atlassian access layer for Claude Code — Jira issues, Bitbucket pipelines, and PR management in one plugin.

## Quick Start

```
/atlassian-extended:setup
```

## Skills

| Skill | Command | Purpose |
|-------|---------|---------|
| **Setup** | `/setup` | Configure Bitbucket + Jira credentials |
| **Jira** | `/jira` | Issue CRUD, search, comments, transitions |
| **Triage** | `/triage` | Bug duplicate detection + new issue creation |
| **Pipeline** | `/pipeline` | Pipeline status, monitoring, failure diagnosis |
| **PR** | `/pr` | PR details, diff, comments, reviews |

## Prerequisites

### Bitbucket REST API

- `BITBUCKET_EMAIL` — Atlassian account email
- `BITBUCKET_API_TOKEN` — App password with `repository:read`, `pullrequest:read`, `pullrequest:write`, `pipeline:read` scopes

Create at: https://bitbucket.org/account/settings/app-passwords/

### Jira MCP

OAuth authentication via Atlassian MCP server. Authenticated on first use.

## Install

```
/plugin marketplace add yoonjong12/atlassian-extended
/plugin install atlassian-extended@atlassian-extended
```

## References

- `references/jira-mcp-tools.md` — Jira MCP tool signatures and JQL patterns
- `references/bitbucket-api.md` — Bitbucket Cloud REST API endpoints
