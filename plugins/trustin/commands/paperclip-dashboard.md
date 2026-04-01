---
description: Show Paperclip company dashboard
allowed-tools: Bash
---

Run the Paperclip dashboard command and present the results clearly:

```bash
pnpm paperclipai dashboard get --json
```

Parse the JSON output and present a readable summary:

1. **Agent status** — show counts by status (active, idle, running, error, paused)
2. **Task breakdown** — show counts by status (todo, in_progress, blocked, done)
3. **Stale tasks** — list any tasks stuck in progress without recent updates
4. **Cost summary** — current month spend vs budget, with burn rate
5. **Recent activity** — last 5 mutations

Highlight anything that needs attention: blocked tasks, agents in error, budget approaching limits.
