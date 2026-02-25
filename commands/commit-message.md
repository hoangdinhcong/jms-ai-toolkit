---
description: Generate commit message from staged changes with context-aware analysis
disable-model-invocation: true
allowed-tools: Bash, Grep, Read, mcp__mcp-server-git__git_diff_staged, mcp__mcp-server-git__git_status, mcp__mcp-server-git__git_log, mcp__mcp-server-git__git_add, mcp__mcp-server-git__git_commit, mcp__linear__get_issue, mcp__linear__list_comments
---

# Generate Commit Message

**FORMATTING: Always use proper markdown - headers, bullet points, blank lines between sections. Never concatenate items on one line.**

## Flow

### 1. Gather Context
- Check conversation history for known issue numbers or prior discussion
- Ask for Linear issue number if not known (or 'no' to skip)
- If provided, fetch issue details for context

### 2. Analyze Staged Changes
- Run `git diff --staged` and `git log --oneline -5`
- Group changes by module (jms-ai, jms-api, jms-web, etc.)
- Identify change type (feat/fix/refactor/chore)

### 3. Present Options
Show two options:

**1) Single Commit** - one message covering all changes

**2) Split Commits** - separate commits per logical unit, showing:
- Commit message for each
- Files included in each

Ask user to pick 1 or 2.

### 4. Commit Loop
For each commit, show the message and files, then ask:

**1) Approve** → `git add <files> && git commit`
**2) Edit** → get new message from user, then commit
**3) Skip** → move to next

### 5. Summary
Show what was committed and what was skipped.

---

## Commit Format

```
#<issue> - <type>(<scope>) <summary>

- Detail 1
- Detail 2
```

**Example:** `#WBS-7326 - feat(inbox) Add notification snooze scheduler`

**Types:** feat, fix, refactor, chore, docs, test, perf

**Rules:**
- First line < 72 chars
- Imperative mood ("Add" not "Added")
- Issue number comes first with #, then dash, then type(scope) summary
- **NO COLON after scope** - use `feat(inbox) Add` NOT `feat(inbox): Add`
