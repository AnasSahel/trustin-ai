---
name: issue-ops
description: >
  Create, assign, track, and manage issues (tasks) in a Paperclip company. Use this skill when
  the user wants to "create an issue", "create a task", "assign a task", "check task status",
  "list issues", "update an issue", "comment on a task", "close a task", "mark as blocked",
  "create subtasks", or anything related to the task lifecycle in Paperclip. Also trigger when
  the user asks about task workflows, status transitions, or work delegation between agents.
version: 0.1.0
---

# Issue Operations

Issues are the unit of work in Paperclip. Every issue traces back to the company goal through
a parent hierarchy.

## Issue Lifecycle

```
backlog → todo → in_progress → in_review → done
                                          ↘ blocked
Terminal states: done, cancelled
```

The transition to `in_progress` requires an **atomic checkout** — only one agent can own a task
at a time. If two agents try to claim the same task simultaneously, one gets a `409 Conflict`.

## CLI Commands

### List Issues

```bash
# All issues
pnpm paperclipai issue list

# Filter by status
pnpm paperclipai issue list --status todo,in_progress

# Filter by assignee
pnpm paperclipai issue list --assignee-agent-id <agent-id>

# Search by text
pnpm paperclipai issue list --match "authentication"
```

### Get Issue Details

```bash
pnpm paperclipai issue get <issue-id-or-identifier>
```

### Create an Issue

```bash
pnpm paperclipai issue create \
  --title "Implement user authentication" \
  --description "Add JWT-based auth with login/logout endpoints" \
  --status todo \
  --priority high
```

When creating issues, always consider:
- **Title** — clear and actionable (verb + object)
- **Description** — enough context for an agent to work autonomously
- **Priority** — `critical`, `high`, `medium`, `low`
- **Parent issue** — set `parentId` to maintain the task hierarchy back to the company goal
- **Goal** — set `goalId` to link the issue to a company goal

### Update an Issue

```bash
pnpm paperclipai issue update <issue-id> \
  --status in_progress \
  --comment "Starting work on this"
```

### Comment on an Issue

```bash
pnpm paperclipai issue comment <issue-id> \
  --body "Found a dependency on the auth module, investigating"

# Reopen a closed issue with a comment
pnpm paperclipai issue comment <issue-id> \
  --body "Reopening — tests revealed a regression" \
  --reopen
```

### Task Checkout / Release

Checkout is how an agent claims exclusive ownership of a task:

```bash
# Checkout (agent claims the task)
pnpm paperclipai issue checkout <issue-id> --agent-id <agent-id>

# Release (agent gives up the task)
pnpm paperclipai issue release <issue-id>
```

**Critical rules:**
- Always checkout before working — never manually PATCH to `in_progress`
- Never retry a `409 Conflict` — the task belongs to someone else, pick a different one
- Always comment on in-progress work before exiting a heartbeat

## Creating Subtask Hierarchies

When decomposing work, create subtasks with proper parent linkage:

```bash
# Parent task
pnpm paperclipai issue create \
  --title "Build authentication system" \
  --priority high

# Subtasks (set parent)
pnpm paperclipai issue create \
  --title "Implement JWT token generation" \
  --description "Create utility for generating and validating JWTs" \
  --priority high
```

Always set `parentId` and `goalId` on subtasks — this maintains the traceable hierarchy back to
the company goal.

## Bulk Operations

For seeding a company with initial work, create issues one at a time via the CLI. For large-scale
imports, use the web UI or API directly.

## Issue Priority

Issues are sorted by priority in list results. Use priority to signal urgency:
- `critical` — blocking the company, needs immediate attention
- `high` — important for current goals
- `medium` — standard work
- `low` — nice to have, backlog material
