---
name: paperclip-company-setup
description: >
  Bootstrap and configure a Paperclip company from scratch. Use this skill when the user wants to
  "create a company", "set up a new AI company", "bootstrap a Paperclip instance", "configure a
  company goal", "create a CEO agent", or anything related to the initial setup of a company in
  Paperclip. Also trigger when the user mentions onboarding, first-time setup, or getting started
  with Paperclip.
version: 0.1.0
---

# Company Setup

Guide the user through creating and configuring a Paperclip company. A company is the top-level
unit in Paperclip — it has a goal, employees (AI agents), org structure, budget, and task hierarchy.

## Prerequisites

Before creating a company, ensure Paperclip is running. If not, bootstrap it:

```bash
pnpm paperclipai run
```

This auto-onboards if config is missing, runs health checks, and starts the server.

## Setup Workflow

### 1. Create the Company

Companies are created through the Paperclip web UI at `http://localhost:3100`. Gather from the user:

- **Company name** — a clear, descriptive name
- **Company goal** — the reason the company exists (e.g., "Build the #1 AI note-taking app at $1M MRR")
- **Monthly budget** (in cents) — the total spend limit for all agents combined

The goal is important because every task in Paperclip traces back to it. It's how agents understand
*why* their work matters. Help the user craft a goal that is specific, measurable, and actionable.

### 2. Create the CEO Agent

Every company needs a root agent — the CEO. This is the top of the org chart. All other agents
report up through this chain.

When creating the CEO:

- **Adapter type** — which LLM runtime to use. See `references/adapters.md` for the full list.
  Common choices: `claude_local` (Claude Code), `codex_local` (OpenAI Codex), `gemini_local` (Gemini CLI).
- **Role** — typically "CEO" with a description of strategic responsibilities
- **Budget** — per-agent monthly spend limit in cents (CEO usually gets a higher budget)
- **Capabilities** — describe what the CEO agent does (e.g., "Strategic planning, task delegation, progress review")

### 3. Build the Org Chart

After the CEO, add more agents as needed. Each agent reports to exactly one manager.
Typical structures:

- **Flat:** CEO + 2-3 IC agents (good for small focused companies)
- **Two-tier:** CEO → department leads → ICs (good for multi-domain work)

For each agent, decide:
- Who they report to
- What adapter they use (can mix adapters — e.g., Claude for coding, Codex for another)
- Their budget allocation
- Their capabilities description

### 4. Set Company Budget

```bash
# View current company details
pnpm paperclipai company get <company-id>
```

Budget is set in the UI or via API. Budget enforcement is automatic:
- 80% → soft alert (agent warned to focus on critical tasks)
- 100% → hard stop (agent auto-paused)

### 5. Create Initial Issues

Seed the company with its first tasks:

```bash
pnpm paperclipai issue create --title "Define strategic plan" \
  --description "Break the company goal into actionable initiatives" \
  --priority high \
  --company-id <company-id>
```

Assign the strategic planning task to the CEO. From there, the CEO will decompose work and
delegate to the org.

### 6. Start Heartbeats

Once everything is configured, trigger the first heartbeat:

```bash
pnpm paperclipai heartbeat run --agent-id <ceo-agent-id>
```

This wakes the CEO, who will check assignments, pick work, and start operating.

## Reference

Read `references/adapters.md` for the full list of adapter types and their configuration options.
