# Linear Integration Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Integrate Linear as the unified task pool with daily planning workflow for Claude Code development.

**Architecture:** Two new skills (`daily-planning`, `picking-a-task`) handle Linear interaction. Two existing skills get minor additions for ticket ID propagation. All skills remain loosely coupled - Linear awareness flows through branch naming convention.

**Tech Stack:** Linear GraphQL API, curl for API calls, existing skill infrastructure

---

## Task 1: Create `daily-planning` Skill

**Files:**
- Create: `skills/daily-planning/SKILL.md`

**Step 1: Create skill directory**

```bash
mkdir -p skills/daily-planning
```

**Step 2: Write SKILL.md with frontmatter and content**

Create `skills/daily-planning/SKILL.md`:

```markdown
---
name: daily-planning
description: "Use when starting your workday or when you need to decide what to work on next. Queries Linear for pending tasks across all teams, ranks by priority/size/deadline, and recommends a focused task list."
---

# Daily Planning

## Overview

Query Linear for pending tasks, rank them by a mixed strategy (priority + deadline + size), and recommend a focused daily task list.

**Announce at start:** "I'm using the daily-planning skill to plan today's tasks."

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
```

**Step 3: Verify file created**

```bash
cat skills/daily-planning/SKILL.md | head -20
```

**Step 4: Commit**

```bash
git add skills/daily-planning/SKILL.md
git commit -m "feat: add daily-planning skill for Linear task management"
```

---

## Task 2: Create `picking-a-task` Skill

**Files:**
- Create: `skills/picking-a-task/SKILL.md`

**Step 1: Create skill directory**

```bash
mkdir -p skills/picking-a-task
```

**Step 2: Write SKILL.md with frontmatter and content**

Create `skills/picking-a-task/SKILL.md`:

```markdown
---
name: picking-a-task
description: "Use when starting work on a specific Linear task. Fetches task details, updates Linear status to In Progress, and guides into the development workflow with ticket ID for branch naming."
---

# Picking a Task

## Overview

Start work on a specific Linear task: fetch details, update status, and guide into the appropriate development workflow.

**Announce at start:** "I'm using the picking-a-task skill to start work on [TICKET-ID]."

## Prerequisites

- Environment variable `LINEAR_API_KEY` must be set
- Valid Linear issue identifier (e.g., PRJ-123)

## The Process

### Step 1: Fetch Task Details

```bash
# Query issue by identifier
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ issue(id: \"ISSUE_ID\") { identifier title description priority estimate dueDate team { name } state { name } comments { nodes { body createdAt } } } }"}' \
  | jq '.data.issue'
```

Note: To query by identifier (e.g., PRJ-123), first search:
```bash
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
```

**Step 3: Verify file created**

```bash
cat skills/picking-a-task/SKILL.md | head -20
```

**Step 4: Commit**

```bash
git add skills/picking-a-task/SKILL.md
git commit -m "feat: add picking-a-task skill for starting Linear tasks"
```

---

## Task 3: Update `using-git-worktrees` Skill

**Files:**
- Modify: `skills/using-git-worktrees/SKILL.md`

**Step 1: Read current file to find insertion point**

Look for the "Creation Steps" section, specifically after "### 2. Create Worktree".

**Step 2: Add Ticket ID section after "## Creation Steps" intro**

Insert after line ~75 (after the "## Creation Steps" heading but before "### 1. Detect Project Name"):

```markdown
### Branch Naming with Ticket ID

If a previous skill (e.g., `picking-a-task`) provided a Ticket ID:

**Branch name MUST follow format:** `TICKET-ID/description`

Examples:
- `PRJ-123/fix-login-timeout`
- `API-42/add-user-avatar`

This enables automatic Linear ↔ GitHub PR linking.

**If no Ticket ID provided:** Use standard descriptive branch name.
```

**Step 3: Verify changes**

```bash
grep -A 10 "Branch Naming with Ticket ID" skills/using-git-worktrees/SKILL.md
```

**Step 4: Commit**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat(using-git-worktrees): add ticket ID branch naming convention"
```

---

## Task 4: Update `finishing-a-development-branch` Skill

**Files:**
- Modify: `skills/finishing-a-development-branch/SKILL.md`

**Step 1: Read current file to find insertion point**

Look for "#### Option 2: Push and Create PR" section (around line 89).

**Step 2: Add auto-link section before the PR creation code block**

Insert after "#### Option 2: Push and Create PR" heading, before the code block:

```markdown
**Auto-link Linear Issue:**

Before creating PR, extract ticket ID from branch name:

```bash
branch=$(git branch --show-current)
ticket_id=$(echo "$branch" | grep -oE '^[A-Z]+-[0-9]+' || true)
```

If ticket ID found, prepend to PR body: `Fixes TICKET-ID`

This triggers Linear to auto-close the issue when PR merges.
```

**Step 3: Update the PR creation code block**

Replace the existing `gh pr create` block with:

```bash
# Push branch
git push -u origin <feature-branch>

# Extract ticket ID from branch name
branch=$(git branch --show-current)
ticket_id=$(echo "$branch" | grep -oE '^[A-Z]+-[0-9]+' || true)

# Build PR body
if [ -n "$ticket_id" ]; then
  fixes_line="Fixes $ticket_id"
else
  fixes_line=""
fi

# Create PR
gh pr create --title "<title>" --body "$(cat <<EOF
## Summary
$fixes_line
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

**Step 4: Verify changes**

```bash
grep -A 20 "Auto-link Linear Issue" skills/finishing-a-development-branch/SKILL.md
```

**Step 5: Commit**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): auto-link Linear issues in PR"
```

---

## Task 5: Test Skills Manually

**Files:**
- None (manual verification)

**Step 1: Verify skill files exist and have correct structure**

```bash
ls -la skills/daily-planning/SKILL.md skills/picking-a-task/SKILL.md
```

**Step 2: Verify frontmatter is valid**

```bash
head -5 skills/daily-planning/SKILL.md
head -5 skills/picking-a-task/SKILL.md
```

**Step 3: Check skills are discoverable**

```bash
# From project root, list all skills
ls skills/*/SKILL.md | wc -l
```

Expected: 16 skills (14 original + 2 new)

**Step 4: Verify modifications to existing skills**

```bash
grep -l "Ticket ID" skills/using-git-worktrees/SKILL.md skills/finishing-a-development-branch/SKILL.md
```

Expected: Both files should match.

---

## Task 6: Update CLAUDE.md with Linear Setup Instructions

**Files:**
- Modify: `CLAUDE.md`

**Step 1: Add Linear configuration section**

Append to CLAUDE.md:

```markdown

## Linear Integration

### Setup

1. Set `LINEAR_API_KEY` environment variable:
   ```bash
   export LINEAR_API_KEY="lin_api_xxx"
   ```

2. Enable Linear GitHub Integration in Linear Settings → Integrations → GitHub

### Workflow

1. `/daily-planning` - Plan today's tasks from Linear backlog
2. `/picking-a-task PRJ-123` - Start work on specific task
3. Branch naming: `PRJ-123/description` (automatic via skills)
4. PR creation: Auto-includes `Fixes PRJ-123` (automatic via skills)
```

**Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add Linear integration setup instructions to CLAUDE.md"
```

---

## Summary

| Task | Description | Files Changed |
|------|-------------|---------------|
| 1 | Create daily-planning skill | +1 new file |
| 2 | Create picking-a-task skill | +1 new file |
| 3 | Update using-git-worktrees | ~10 lines added |
| 4 | Update finishing-a-development-branch | ~15 lines added |
| 5 | Manual verification | None |
| 6 | Update CLAUDE.md | ~20 lines added |

**Total commits:** 6
**New files:** 2
**Modified files:** 3
