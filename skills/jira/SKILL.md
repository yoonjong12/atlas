---
name: jira
description: "Access Jira issues — read, search, create, edit, comment, and transition. This skill should be used when the user asks to 'check an issue', 'look up a story', 'create a subtask', 'search jira', 'add a comment', 'change status', 'transition issue', or references a Jira issue key (e.g., WAO-372). Trigger on: '이슈 확인', '스토리 확인', '서브태스크', '이슈 만들어', '코멘트', '상태 변경', 'JQL', 'jira search'"
argument-hint: "<issue key, JQL query, or description>"
---

# Jira — Issue Access Layer

Read, search, create, edit, comment, and transition Jira issues via Atlassian MCP.

## Tool Discovery

MCP tools are deferred. Load required tools before first use:

```
ToolSearch({ query: "+atlassian getJiraIssue" })
ToolSearch({ query: "+atlassian searchJiraIssuesUsingJql" })
```

Load write tools only when needed:

```
ToolSearch({ query: "+atlassian createJiraIssue" })
ToolSearch({ query: "+atlassian editJiraIssue" })
ToolSearch({ query: "+atlassian addCommentToJiraIssue" })
ToolSearch({ query: "+atlassian transitionJiraIssue" })
```

## cloudId

All MCP tools require `cloudId` — the Atlassian site URL (e.g., `your-org.atlassian.net`). Discover the correct value via `atlassianUserInfo({})` after authentication.

## Read Operations

### Single Issue Lookup

When the user provides an issue key (e.g., `WAO-372`):

```typescript
getJiraIssue({
  cloudId: "mindai.atlassian.net",
  issueIdOrKey: "WAO-372",
  fields: ["summary", "description", "status", "assignee", "subtasks", "parent", "priority"],
  responseContentFormat: "markdown"
})
```

Present results concisely: key, summary, status, assignee, and subtask list if any.

### Search with JQL

When the user asks to find or list issues:

```typescript
searchJiraIssuesUsingJql({
  cloudId: "mindai.atlassian.net",
  jql: "project = WAO AND type = Story AND status != Done ORDER BY updated DESC",
  maxResults: 10,
  fields: ["summary", "status", "assignee"],
  responseContentFormat: "markdown"
})
```

Translate natural language queries to JQL:
- "WAO 프로젝트 진행중인 스토리" → `project = WAO AND type = Story AND status = "In Progress"`
- "내 할당된 이슈" → `assignee = currentUser() AND status != Done`
- "이번주 생성된 이슈" → `project = WAO AND created >= startOfWeek()`
- "에러 핸들링 관련" → `project = WAO AND text ~ "error handling"`

For detailed JQL patterns, consult `references/jira-mcp-tools.md`.

### Hierarchy Navigation

To show an issue's full context (Epic → Story → Subtask):

1. Fetch the target issue with `parent` and `subtasks` fields
2. If it has a parent, fetch the parent for context
3. If it has subtasks, they are included in the response

## Write Operations

### Create Issue

```typescript
createJiraIssue({
  cloudId: "mindai.atlassian.net",
  projectKey: "WAO",
  issueTypeName: "Subtask",    // "Epic", "Story", "Subtask", "Task", "Bug"
  summary: "Issue title",
  description: "Description in markdown",
  contentFormat: "markdown",
  parent: "WAO-372"            // required for Story under Epic, Subtask under Story
})
```

**Important**: Use `"Subtask"` not `"Sub-task"` for issueTypeName.

### Edit Issue

```typescript
editJiraIssue({
  cloudId: "mindai.atlassian.net",
  issueIdOrKey: "WAO-372",
  fields: { "summary": "Updated title" },
  contentFormat: "markdown"
})
```

### Add Comment

```typescript
addCommentToJiraIssue({
  cloudId: "mindai.atlassian.net",
  issueIdOrKey: "WAO-372",
  commentBody: "Comment text",
  contentFormat: "markdown"
})
```

### Transition (Status Change)

Two-step process:

1. Get available transitions:
   ```typescript
   getTransitionsForJiraIssue({
     cloudId: "mindai.atlassian.net",
     issueIdOrKey: "WAO-372"
   })
   ```

2. Execute transition using the transition ID from step 1:
   ```typescript
   transitionJiraIssue({
     cloudId: "mindai.atlassian.net",
     issueIdOrKey: "WAO-372",
     transition: { id: "31" }
   })
   ```

## Token Efficiency

- Request only needed fields via `fields` array
- Use `maxResults` aggressively (5 for selection, 1 for latest)
- Use `responseContentFormat: "markdown"` for compact output

## Additional Resources

- **`references/jira-mcp-tools.md`** — Full tool signatures, JQL patterns, metadata tools
