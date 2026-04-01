---
name: paperclip-diagnostics
description: >
  Diagnose and fix Paperclip instance issues — health checks, heartbeat debugging, adapter
  troubleshooting, and environment validation. Use this skill when the user says "paperclip is
  broken", "agent is stuck", "heartbeat failed", "run doctor", "check health", "something is
  wrong with paperclip", "agent won't start", "409 error", "agent error state", or anything
  related to troubleshooting Paperclip. Also trigger when the user mentions environment config,
  connection issues, or server problems.
version: 0.1.0
---

# Diagnostics

Tools and procedures for troubleshooting Paperclip instances and agents.

## Health Checks

### Run Doctor

```bash
# Check health
pnpm paperclipai doctor

# Check and auto-repair
pnpm paperclipai doctor --repair
```

Doctor validates:
- Server configuration
- Database connectivity
- Secrets adapter configuration
- Storage configuration
- Missing key files

### Show Environment

```bash
pnpm paperclipai env
```

Displays resolved environment configuration — useful for verifying what Paperclip sees.

## Common Problems

### Agent Stuck in Error State

1. Check agent details: `pnpm paperclipai agent get <agent-id>`
2. Check recent activity: `pnpm paperclipai activity list --agent-id <agent-id>`
3. Look at the run history in the web UI for error details
4. Common causes:
   - Adapter not installed or misconfigured (e.g., Claude Code CLI not on PATH)
   - API key expired or missing
   - Agent exceeded budget (check cost-control skill)

### 409 Conflict on Task Checkout

This means another agent already owns the task. This is *expected behavior* — Paperclip uses
atomic checkout to prevent two agents from working on the same thing.

**Resolution:** Do not retry. Pick a different task. If the task appears abandoned (the owning
agent is stuck), the board operator can release it:

```bash
pnpm paperclipai issue release <issue-id>
```

### Heartbeat Not Firing

1. Check if the agent is paused: `pnpm paperclipai agent get <agent-id>` → look at status
2. If paused due to budget: increase budget or wait for month reset
3. If manually paused: resume from the web UI
4. Check scheduler configuration — heartbeats can be schedule-triggered, and a misconfigured
   schedule means no wakes

### Manual Heartbeat Trigger

Force a heartbeat for debugging:

```bash
pnpm paperclipai heartbeat run --agent-id <agent-id>
```

This wakes the agent immediately, regardless of schedule.

### Server Won't Start

```bash
# One-command fix attempt
pnpm paperclipai run --data-dir ./tmp/paperclip-dev

# Or step by step:
pnpm paperclipai doctor --repair
pnpm paperclipai run
```

Check logs at `~/.paperclip/instances/default/logs`.

### Database Issues

Paperclip uses embedded PostgreSQL (PGlite) by default — no external DB needed. If using an
external PostgreSQL:

1. Check connectivity: `pnpm paperclipai doctor`
2. Verify config: `pnpm paperclipai env`
3. Re-configure: `pnpm paperclipai configure --section server`

## Instance Management

### Multiple Instances

Run isolated instances by specifying a data directory:

```bash
pnpm paperclipai run --data-dir ./tmp/paperclip-dev
pnpm paperclipai doctor --data-dir ./tmp/paperclip-dev
```

Or use environment variables:

```bash
PAPERCLIP_HOME=/custom/home PAPERCLIP_INSTANCE_ID=dev pnpm paperclipai run
```

### Reconfigure

```bash
pnpm paperclipai configure --section server
pnpm paperclipai configure --section secrets
pnpm paperclipai configure --section storage
```

### Allow Private Hostnames

For Tailscale or other private network access:

```bash
pnpm paperclipai allowed-hostname my-tailscale-host
```

## API Error Reference

| Code | Meaning | What to Do |
|------|---------|-----------|
| 400 | Validation error | Check request body fields |
| 401 | Unauthenticated | API key missing or invalid |
| 403 | Unauthorized | No permission for this action |
| 404 | Not found | Entity doesn't exist or wrong company |
| 409 | Conflict | Another agent owns the task — do not retry |
| 422 | Semantic violation | Invalid state transition (e.g., backlog → done) |
| 500 | Server error | Transient failure, comment on task and move on |
