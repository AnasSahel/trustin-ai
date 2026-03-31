# TrustIn Paperclip Plugin

Operate [Paperclip](https://docs.paperclip.ing) — the control plane for autonomous AI companies — directly from Claude.

## What it does

This plugin gives Claude the knowledge and commands to help you manage Paperclip companies, agents, issues, approvals, budgets, and diagnostics via the `paperclipai` CLI.

## Skills

| Skill | What it covers |
|-------|---------------|
| `trustin-paperclip:company-setup` | Bootstrap a company: create, set goals, configure CEO, set budgets |
| `trustin-paperclip:agent-ops` | Hire, configure, organize, and monitor AI agents |
| `trustin-paperclip:issue-ops` | Create, assign, track, and manage the task lifecycle |
| `trustin-paperclip:board-ops` | Dashboard monitoring, approvals, activity audit, governance |
| `trustin-paperclip:cost-control` | Budget management, spend monitoring, cost analysis |
| `trustin-paperclip:diagnostics` | Health checks, heartbeat debugging, troubleshooting |

## Commands

| Command | Description |
|---------|-------------|
| `/paperclip-dashboard` | Quick company health overview |
| `/paperclip-status` | Agent statuses + open issues summary |
| `/paperclip-approve` | List and handle pending approvals |
| `/paperclip-doctor` | Run health checks with auto-repair |

## Prerequisites

- Paperclip installed and running locally (`pnpm paperclipai run`)
- Node.js 20+ and pnpm 9+

## Usage

Skills trigger automatically when you ask about Paperclip operations. Commands are available as slash commands.

Examples:
- "Create a new company for building an AI writing assistant"
- "Show me the dashboard"
- "Hire a backend developer agent using Claude"
- "What's the budget status?"
- "Something is wrong with the CEO agent, it's stuck"
- `/paperclip-dashboard`
- `/paperclip-approve`
