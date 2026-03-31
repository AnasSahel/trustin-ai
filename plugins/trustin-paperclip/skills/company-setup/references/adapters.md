# Adapter Reference

Adapters connect Paperclip to agent runtimes. Each agent has one adapter type.

## Built-in Adapters

| Adapter | Type Key | What it runs |
|---------|----------|-------------|
| Claude Local | `claude_local` | Claude Code CLI locally |
| Codex Local | `codex_local` | OpenAI Codex CLI locally |
| Gemini Local | `gemini_local` | Gemini CLI locally |
| OpenCode Local | `opencode_local` | OpenCode CLI (multi-provider) |
| Cursor Local | `cursor` | Cursor CLI locally |
| OpenClaw Gateway | `openclaw_gateway` | Sends wake payloads to an OpenClaw gateway |
| Hermes Local | `hermes_local` | Hermes Agent CLI locally |
| Pi Local | `pi_local` | Pi CLI locally |
| Process | `process` | Arbitrary shell commands |
| HTTP | `http` | Webhooks to external agents |

## Choosing an Adapter

- **Coding agent?** → `claude_local`, `codex_local`, `gemini_local`, or `opencode_local`
- **Run a script or command?** → `process`
- **Call an external service?** → `http`
- **Custom runtime?** → Create a custom adapter

## Adapter Architecture

Each adapter is a package with three modules:

- **Server module** — `execute()` function that spawns the agent + environment diagnostics
- **UI module** — stdout parser for the run viewer + config form fields
- **CLI module** — terminal formatter for `paperclipai run --watch`

Adapters are adapter-agnostic from the control plane's perspective — any runtime that can call
Paperclip's REST API works as an agent.
