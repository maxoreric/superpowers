---
name: daily-planning
description: "Use when starting your workday, planning what to work on next, or deciding which Linear task to pick up"
---

# Daily Planning

## Overview

Query Linear for pending tasks, rank them by a mixed strategy (priority + deadline + size), and recommend a focused daily task list.

**Announce at start:** "I'm using the daily-planning skill to plan today's tasks."

## When to Use

- Starting your workday and need to prioritize tasks
- Returning from a break and unsure what to work on
- Finishing a task and deciding what comes next
- Weekly planning to identify high-priority items

## Prerequisites

- Environment variable `LINEAR_API_KEY` must be set
- Linear workspace with teams configured

## The Process

### Step 1: Query Linear for Pending Tasks

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ issues(filter: {state: {type: {in: [\"backlog\", \"unstarted\"]}}}, first: 50) { nodes { identifier title priority dueDate estimate team { name } state { name } } } }"}' \
  | jq '.data.issues.nodes'
```

### Step 2: Calculate Priority Score

For each task, calculate composite score:

| Factor | Score |
|--------|-------|
| Priority: Urgent | +4 |
| Priority: High | +3 |
| Priority: Medium | +2 |
| Priority: Low/None | +1 |
| Due: Overdue | +3 |
| Due: Within 3 days | +2 |
| Due: Within 7 days | +1 |
| Due: None/Later | +0 |
| Size: S (estimate 1-2) | +0 |
| Size: M (estimate 3-5) | -1 |
| Size: L (estimate 8+) | -2 |

**Composite Score = Priority + Due Date - Size Penalty**

### Step 3: Present Top Tasks

Display sorted list (highest score first):

```
Today's recommended tasks:

| #  | ID       | Title                    | Pri | Size | Due     | Score |
|----|----------|--------------------------|-----|------|---------|-------|
| 1  | PRJ-123  | Fix login timeout        | Urgent | S | Today   | 7     |
| 2  | PRJ-456  | Add avatar upload        | High   | M | 3 days  | 4     |
| 3  | PRJ-789  | Refactor DB pool         | Medium | L | -       | 0     |

Select tasks for today (enter numbers, e.g., "1, 2"):
```

### Step 4: Confirm and Handoff

After user selects:

```
Today's tasks confirmed:
1. PRJ-123: Fix login timeout (S)
2. PRJ-456: Add avatar upload (M)

Start first task with: /picking-a-task PRJ-123
```

## Error Handling

**If LINEAR_API_KEY not set:**
```
Error: LINEAR_API_KEY environment variable not set.

To set up:
1. Go to Linear Settings → API → Create Key
2. Add to your shell profile: export LINEAR_API_KEY="lin_api_xxx"
```

**If API returns error:**
```
Error querying Linear API: [error message]

Check:
- API key is valid
- Network connectivity
- Linear service status
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| No pending tasks | Report "No pending tasks in Linear" |
| API error | Show error, suggest fixes |
| User selects 0 tasks | Confirm "No tasks selected for today" |
| User selects 5+ tasks | Warn "Consider focusing on fewer tasks" |

## Integration

**Pairs with:**
- **picking-a-task** - Start working on selected task
- **brainstorming** - For M/L tasks that need design discussion
