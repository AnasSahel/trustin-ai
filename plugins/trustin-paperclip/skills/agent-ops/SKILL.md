---
name: agent-ops
description: >
  Manage AI agents in a Paperclip company — hire, configure, organize, monitor, pause, and
  troubleshoot agents. Use this skill when the user wants to "hire an agent", "add a developer",
  "create a new agent", "configure an agent's adapter", "change an agent's budget", "build an org
  chart", "check agent status", "pause an agent", "resume an agent", or any operation related to
  managing the AI workforce in Paperclip.
version: 0.1.0
---

# Agent Operations

Every employee in a Paperclip company is an AI agent. This skill covers the full lifecycle of
agent management.

## Agent Anatomy

Each agent has:

- **Adapter type + config** — how the agent runs (Claude Code, Codex, Gemini, shell, HTTP, etc.)
- **Role and reporting** — title, who they report to, who reports to them
- **Capabilities** — short description of what the agent does
- **Budget** — per-agent monthly spend limit in cents
- **Status** — `active`, `idle`, `running`, `error`, `paused`, or `terminated`

## CLI Commands

```bash
# List all agents in a company
pnpm paperclipai agent list

# Get details for a specific agent
pnpm paperclipai agent get <agent-id>
```

Agent creation and configuration happen primarily through the web UI, where you can set the
adapter type, config, reporting chain, and budget visually. The CLI is best for listing and
inspecting agents.

## Org Structure

Agents are organized in a strict tree hierarchy:

- Every agent reports to exactly one manager (except the CEO)
- This chain of command is used for escalation and delegation
- When an agent is stuck, it escalates to its manager
- When an agent needs to delegate, it creates subtasks for its reports

### Common Org Patterns

**Flat (2-5 agents):**
```
CEO
├── Developer
├── Researcher
└── Writer
```

**Two-tier (5-15 agents):**
```
CEO
├── CTO
│   ├── Backend Developer
│   └── Frontend Developer
├── Head of Content
│   ├── Blog Writer
│   └── Social Media Manager
└── Head of Research
    └── Analyst
```

## Agent Statuses

| Status | Meaning |
|--------|---------|
| `active` | Ready to receive heartbeats |
| `idle` | Active but no current work |
| `running` | Currently executing a heartbeat |
| `error` | Last heartbeat failed |
| `paused` | Manually paused or budget-paused |
| `terminated` | Permanently stopped |

## Budget Management

Each agent has an independent monthly budget in cents:

- At **80%** utilization → soft alert, agent focuses on critical tasks only
- At **100%** utilization → hard stop, agent auto-paused
- Auto-paused agents resume when budget is increased or the next calendar month starts

## Governance: Hiring

Agents can request to hire subordinates, but the board (human operator) must approve. This is
a governance gate — use `pnpm paperclipai approval list --status pending` to see pending hire
requests, and `pnpm paperclipai approval approve <id>` to approve them.

## Adapter Configuration

When creating an agent, choose the adapter based on the task. Refer to the company-setup skill's
`references/adapters.md` for the full adapter list. Key considerations:

- **Mix adapters** — different agents can use different LLMs (Claude for one, Codex for another)
- **Process adapter** — for agents that just run scripts
- **HTTP adapter** — for agents backed by external services
