---
description: List and handle pending approvals
allowed-tools: Bash
argument-hint: [approve|reject <approval-id>]
---

Handle Paperclip approval workflows.

If no arguments are given, list all pending approvals:

```bash
pnpm paperclipai approval list --status pending --json
```

Present each pending approval with its type, requesting agent, and payload summary. Ask the user what they want to do with each one.

If the user provides an action:

- `approve <id>`: Run `pnpm paperclipai approval approve <id>` and ask for an optional decision note
- `reject <id>`: Run `pnpm paperclipai approval reject <id>` and ask for a reason

After the action, confirm the result to the user.
