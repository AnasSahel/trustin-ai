---
name: board-ops
description: >
  Board operator workflows for Paperclip — dashboard monitoring, approval handling, activity
  audit, and governance. Use this skill when the user wants to "check the dashboard", "review
  approvals", "approve a hire", "reject an approval", "see recent activity", "check what agents
  are doing", "review pending requests", "audit the company", or any board-level oversight of
  a Paperclip company. Also trigger when the user asks about governance, monitoring, or
  operational health.
version: 0.1.0
---

# Board Operations

The board operator (you, the human) has full visibility and control over the Paperclip company.
Every mutation is logged in an activity audit trail.

## Dashboard

Get a real-time overview of company health:

```bash
pnpm paperclipai dashboard get
```

The dashboard shows:
- **Agent status** — how many agents are active, idle, running, or in error
- **Task breakdown** — counts by status (todo, in progress, blocked, done)
- **Stale tasks** — tasks stuck in progress too long without updates
- **Cost summary** — current month spend vs budget, burn rate
- **Recent activity** — latest mutations across the company

### What to Watch For

- **Blocked tasks** — read comments to understand what's blocking, then take action (reassign,
  unblock, or approve)
- **Budget utilization** — agents auto-pause at 100%. If approaching 80%, consider increasing
  budget or reprioritizing work
- **Stale work** — tasks in progress with no recent comments may indicate a stuck agent. Check
  the agent's run history for errors

## Approvals

Some actions require board approval. This is a governance gate.

### List Pending Approvals

```bash
pnpm paperclipai approval list --status pending
```

### Get Approval Details

```bash
pnpm paperclipai approval get <approval-id>
```

### Approve / Reject / Request Revision

```bash
# Approve
pnpm paperclipai approval approve <approval-id> \
  --decision-note "Looks good, proceed"

# Reject
pnpm paperclipai approval reject <approval-id> \
  --decision-note "Budget too high, propose a lower amount"

# Request revision
pnpm paperclipai approval request-revision <approval-id> \
  --decision-note "Please clarify the scope before I approve"
```

### Resubmit / Comment

```bash
# Resubmit after revision
pnpm paperclipai approval resubmit <approval-id> \
  --payload '{"name": "revised-agent", "budgetMonthlyCents": 3000}'

# Comment on an approval
pnpm paperclipai approval comment <approval-id> \
  --body "Can you add more detail on the expected output?"
```

### Approval Types

- **Hiring agents** — agents can request to hire subordinates, board must approve
- **CEO strategy** — the CEO's initial strategic plan requires board approval
- **Board overrides** — the board can pause, resume, or terminate any agent and reassign any task

## Activity Audit

Every action in Paperclip is logged:

```bash
# All recent activity
pnpm paperclipai activity list

# Filter by agent
pnpm paperclipai activity list --agent-id <agent-id>

# Filter by entity
pnpm paperclipai activity list --entity-type issue --entity-id <issue-id>
```

Use the activity log to understand what happened, who did it, and when. This is your audit trail
for accountability.

## Governance Powers

As board operator, you can:
- **Pause any agent** — stops heartbeats immediately
- **Resume any agent** — restarts heartbeats
- **Terminate any agent** — permanent stop
- **Reassign any task** — move work between agents
- **Override any status** — unblock, cancel, or reopen tasks
