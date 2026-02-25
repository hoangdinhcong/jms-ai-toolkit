---
description: Complete bug fix workflow - commit, comment, and update status
disable-model-invocation: true
allowed-tools: mcp__linear__get_issue, mcp__linear__create_comment, mcp__linear__update_issue, mcp__linear__list_issue_statuses, mcp__linear__get_issue_status, mcp__mcp-server-git__git_status, mcp__mcp-server-git__git_add, mcp__mcp-server-git__git_commit, mcp__mcp-server-git__git_diff_unstaged, mcp__mcp-server-git__git_diff_staged
---

# Complete Fix Workflow

Complete the bug fix by committing, documenting, and updating Linear status.

Usage: `/complete-fix [issue-id]`

## Workflow Steps:

### 1. Create Summary & Understand Context
First, I'll review our chat history to understand:
- Whether `/fix-bug` was run previously for this issue
- The root cause identified during investigation
- The solution implemented
- Get issue details from Linear: $ARGUMENTS

Then I'll ask you:
- Is there any additional context about the root cause?
- Any specific details about the solution to highlight?
- Any other information to include in the summary?

Based on the chat history and your input, I'll generate a concise summary with:
- **Root Cause**: Brief explanation of what was causing the bug
- **Solution**: Brief explanation of how it was fixed
- **Impact**: What areas/features are affected by this fix
- **Verification**: How to verify the fix is working correctly

Present this summary and wait for your confirmation.

### 2. Prepare Commit
- Extract issue number and title from Linear
- Create commit message using ONLY this format: `#[issue-number] 🐛 [issue-title]`
- CRITICAL: Use ONLY the exact format above - NO additional text, descriptions, or metadata
- CRITICAL: Do NOT add "Co-Authored-By: Claude <noreply@anthropic.com>"
- CRITICAL: Do NOT add "🤖 Generated with [Claude Code](https://claude.ai/code)"
- CRITICAL: Do NOT add any other Claude-generated metadata or signatures
- Review changes using git status and git diff
- Identify files related to the bug fix
- If unsure which files are related, I'll ask you to confirm which files should be included in the commit

### 3. Create Git Commit
- Add only the related files to staging (not all modified files)
- Use git MCP to commit with the prepared message
- CRITICAL: Ensure the commit message contains ONLY: `#[issue-number] 🐛 [issue-title]`
- CRITICAL: No Claude signatures, co-author tags, or generated metadata in the commit

### 4. Update Linear Issue
After confirmation:
- Add the summary as a comment on the Linear issue with this format:
  ```
  **Root Cause:**
  [Explanation of what was causing the bug]

  **Solution:**
  [Explanation of how it was fixed]

  **Impact:**
  [What areas/features are affected by this fix]

  **Verification:**
  [How to verify the fix is working correctly]
  ```

### 5. Update Issue Status
- Find the "Dev Complete" status in the team
- Update the Linear issue status to "Dev Complete"