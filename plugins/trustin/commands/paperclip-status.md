---
description: Show agent statuses and open issues
allowed-tools: Bash
---

Get a quick status overview of the Paperclip company. Run both commands:

```bash
pnpm paperclipai agent list --json
pnpm paperclipai issue list --status todo,in_progress,blocked --json
```

Present the results as:

1. **Agents** — table showing each agent's name, role, status, and budget utilization
2. **Open issues** — table showing title, status, priority, and assignee for all non-done issues

Flag any problems: agents in error state, tasks blocked with no recent comments, unassigned high-priority issues.
