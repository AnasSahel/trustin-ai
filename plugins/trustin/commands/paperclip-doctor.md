---
description: Run Paperclip health checks with auto-repair
allowed-tools: Bash
---

Run Paperclip health diagnostics:

```bash
pnpm paperclipai doctor --repair
```

Parse the output and present:

1. **Checks passed** — list what's healthy
2. **Issues found** — list what failed and whether auto-repair fixed it
3. **Manual action needed** — if anything couldn't be auto-repaired, explain what the user needs to do

If everything passes, confirm the instance is healthy.
