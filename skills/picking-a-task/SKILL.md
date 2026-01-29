---
name: picking-a-task
description: "Use when starting work on a specific Linear task, after selecting from daily-planning or when given a ticket ID directly"
---

# Picking a Task

## Overview

Start work on a specific Linear task: fetch details, update status, and guide into the appropriate development workflow.

**Announce at start:** "I'm using the picking-a-task skill to start work on [TICKET-ID]."

## When to Use

- After daily-planning has recommended tasks and you've selected one
- When given a specific ticket ID to work on (e.g., "start PRJ-123")
- Resuming work on a previously started task

## Prerequisites

- Environment variable `LINEAR_API_KEY` must be set
- Valid Linear issue identifier (e.g., PRJ-123)

## The Process

### Step 1: Fetch Task Details

```bash
# Query issue by identifier (search first to get ID)
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ issueSearch(query: \"PRJ-123\", first: 1) { nodes { id identifier title description priority estimate dueDate team { name } state { name } } } }"}' \
  | jq '.data.issueSearch.nodes[0]'
```

### Step 2: Display Task Summary

```
┌─────────────────────────────────────────┐
│ PRJ-123: Fix login timeout bug          │
├─────────────────────────────────────────┤
│ Priority: Urgent       Size: S          │
│ Team: API              Due: Today       │
├─────────────────────────────────────────┤
│ Description:                            │
│ Users report intermittent 504 errors    │
│ during login. Suspect Redis connection  │
│ pool exhaustion under load...           │
└─────────────────────────────────────────┘
```

### Step 3: Update Linear Status

```bash
# Get workflow state ID for "In Progress"
STATE_ID=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ workflowStates(filter: {name: {eq: \"In Progress\"}}) { nodes { id name } } }"}' \
  | jq -r '.data.workflowStates.nodes[0].id')

# Update issue state
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"query\": \"mutation { issueUpdate(id: \\\"ISSUE_ID\\\", input: {stateId: \\\"$STATE_ID\\\"}) { success } }\"}"
```

Report: "Updated Linear status to In Progress"

### Step 4: Choose Next Step

```
Task ready. Next step:

a) Start implementation directly (recommended for S tasks)
b) Brainstorm design first (recommended for M/L tasks)
c) View more details (comments, related issues)

Which option?
```

### Step 5: Handoff with Ticket ID

**Critical instruction for subsequent skills:**

```
IMPORTANT: When creating a branch (via using-git-worktrees or manually),
the branch name MUST start with: PRJ-123/

Example: PRJ-123/fix-login-timeout

This enables automatic Linear ↔ GitHub linking.
```

**If option (a):** Guide to `using-git-worktrees` with ticket ID prefix requirement.

**If option (b):** Start `brainstorming` skill, remind about ticket ID for later.

## Error Handling

**If issue not found:**
```
Error: Issue PRJ-123 not found in Linear.

Check:
- Issue identifier is correct
- You have access to this team's issues
```

**If status update fails:**
```
Warning: Could not update Linear status.
Continuing with task. Update status manually if needed.
```

## Quick Reference

| Task Size | Recommended Path |
|-----------|------------------|
| S (1-2 pts) | Direct implementation |
| M (3-5 pts) | Consider brainstorming |
| L (8+ pts) | Brainstorming recommended |

## Integration

**Called by:**
- **daily-planning** - After selecting today's tasks

**Pairs with:**
- **brainstorming** - For tasks needing design discussion
- **using-git-worktrees** - Creates branch with ticket ID prefix
