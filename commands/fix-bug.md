---
description: Fix bug from Linear issue with guided workflow
disable-model-invocation: true
allowed-tools: mcp__linear__get_issue, mcp__linear__list_comments, Grep, Read, MultiEdit, Edit, WebSearch
---

# Fix Bug Workflow

Fix a bug from Linear issue following a systematic workflow.

Usage: `/fix-bug [issue-id]`

## Workflow Steps:

### 1. Get Issue Details
First, I'll retrieve the Linear issue details and all comments to understand the bug.

```
Use Linear MCP to get issue: $ARGUMENTS
- Get issue details including description, steps to reproduce, acceptance criteria
- Get all comments on the issue
- Analyze the bug report thoroughly
```

### 2. Gather Additional Information
After understanding the bug from Linear, I'll ask you:
- Are there any additional steps to reproduce?
- Do you have any related files or components to investigate?
- What investigation have you done so far?
- Any error logs or screenshots?

### 3. Find Root Cause
Based on the information gathered:
- Search the codebase for relevant files
- Analyze the code flow
- Identify the root cause of the bug
- **Present my findings with confidence level (e.g., "90% confident this is the root cause because...")**
- Ask for your confirmation before proceeding

### 4. Propose Solution
Once root cause is confirmed:
- Develop a solution approach
- Consider edge cases and potential impacts
- **Present the solution with confidence level (e.g., "85% confident this solution will work because...")**
- Ask for your approval before implementation

### 5. Implement Fix
After solution approval:
- Implement the fix
- Make necessary code changes
- Ensure the fix addresses the acceptance criteria

### 6. Code Review
After implementation:
- Review the implemented changes
- Check for:
  - Code quality and consistency with existing patterns
  - Potential side effects or regressions
  - Proper error handling
  - Performance implications
- Present a summary of changes made
- Note: Build and lint checks will be handled separately by you