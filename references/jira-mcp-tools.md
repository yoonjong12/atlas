# Jira MCP Tool Reference

Quick reference for Atlassian MCP tools. All tools require `cloudId` (e.g., `mindai.atlassian.net`).

## Tool Discovery

All Atlassian MCP tools are deferred. Load before first use:

```
ToolSearch({ query: "+atlassian getJiraIssue" })
ToolSearch({ query: "+atlassian searchJiraIssuesUsingJql" })
ToolSearch({ query: "+atlassian createJiraIssue" })
ToolSearch({ query: "+atlassian editJiraIssue" })
ToolSearch({ query: "+atlassian addCommentToJiraIssue" })
ToolSearch({ query: "+atlassian transitionJiraIssue" })
ToolSearch({ query: "+atlassian getTransitionsForJiraIssue" })
ToolSearch({ query: "+atlassian authenticate" })
```

## Read Operations

### getJiraIssue

```typescript
getJiraIssue({
  cloudId: "mindai.atlassian.net",
  issueIdOrKey: "WAO-180",
  fields: ["summary", "description", "status", "assignee", "subtasks"],
  responseContentFormat: "markdown"
})
```

### searchJiraIssuesUsingJql

```typescript
searchJiraIssuesUsingJql({
  cloudId: "mindai.atlassian.net",
  jql: "project = WAO AND type = Story AND status != Done",
  maxResults: 10,
  fields: ["summary", "status", "assignee"],
  responseContentFormat: "markdown"
})
```

### lookupJiraAccountId

```typescript
lookupJiraAccountId({
  cloudId: "mindai.atlassian.net",
  searchString: "yoonjong"
})
```

### atlassianUserInfo

```typescript
atlassianUserInfo({})
```

## Write Operations

### createJiraIssue

```typescript
createJiraIssue({
  cloudId: "mindai.atlassian.net",
  projectKey: "WAO",
  issueTypeName: "Story",       // "Epic", "Story", "Subtask", "Task", "Bug"
  summary: "Issue title",
  description: "Description in Markdown",
  contentFormat: "markdown",
  parent: "WAO-180",            // optional — for Story under Epic, Subtask under Story
  additional_fields: {           // optional
    "priority": {"name": "1"},
    "duedate": "2026-02-10",
    "assignee_account_id": "..."
  }
})
```

**Note**: Use `"Subtask"` not `"Sub-task"` for issueTypeName.

### editJiraIssue

```typescript
editJiraIssue({
  cloudId: "mindai.atlassian.net",
  issueIdOrKey: "WAO-180",
  fields: {
    "summary": "Updated title",
    "description": "Updated description"
  },
  contentFormat: "markdown"
})
```

### addCommentToJiraIssue

```typescript
addCommentToJiraIssue({
  cloudId: "mindai.atlassian.net",
  issueIdOrKey: "WAO-180",
  commentBody: "Comment in Markdown",
  contentFormat: "markdown"
})
```

### transitionJiraIssue

```typescript
// First, get available transitions
getTransitionsForJiraIssue({
  cloudId: "mindai.atlassian.net",
  issueIdOrKey: "WAO-180"
})

// Then transition
transitionJiraIssue({
  cloudId: "mindai.atlassian.net",
  issueIdOrKey: "WAO-180",
  transition: { id: "31" }
})
```

### createIssueLink

```typescript
createIssueLink({
  cloudId: "mindai.atlassian.net",
  inwardIssue: "WAO-181",
  outwardIssue: "WAO-180",
  type: "Blocks"
})
```

## Metadata

### getVisibleJiraProjects

```typescript
getVisibleJiraProjects({
  cloudId: "mindai.atlassian.net"
})
```

### getJiraProjectIssueTypesMetadata

```typescript
getJiraProjectIssueTypesMetadata({
  cloudId: "mindai.atlassian.net",
  projectIdOrKey: "WAO"
})
```

## Common JQL Patterns

```jql
# Active epics
project = WAO AND type = Epic AND status IN (Open, "In Progress") ORDER BY updated DESC

# Stories under epic
parent = WAO-180 AND type = Story

# Subtasks under story
parent = WAO-181 AND type = Subtask

# My incomplete tasks
assignee = currentUser() AND status != Done AND resolution = Unresolved

# Recent issues
project = WAO AND created >= -7d ORDER BY created DESC

# Unassigned stories
project = WAO AND type = Story AND assignee is EMPTY

# By text search
project = WAO AND text ~ "error handling"

# By label
project = WAO AND labels = "backend"

# By sprint
project = WAO AND sprint in openSprints()
```

## Token Efficiency

1. **Selective fields**: Only request needed fields
   ```typescript
   fields: ["summary", "status"]  // not all fields
   ```

2. **Limit results**: Use `maxResults` aggressively
   ```typescript
   maxResults: 5   // for selection lists
   maxResults: 1   // for "latest" queries
   ```

3. **Markdown format**: Use `responseContentFormat: "markdown"` for readable output

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 401/403 | OAuth expired | Re-authenticate via `authenticate` tool |
| 404 | Issue not found or no permission | Verify issue key |
| 400 | Missing required fields | Check issue type metadata |
| "No such tool" | Tool is deferred | Load via ToolSearch first |
