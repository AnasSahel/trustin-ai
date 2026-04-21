# workflow

Workflow slash commands for engineers — terraform, GitHub, Dokploy.

## Contents

- `/terraform-plan-review` — run `terraform plan`, parse the JSON, produce a risk-tiered summary (🔴 destroy / 🟠 replace / 🟡 update / 🟢 safe). Flags SailPoint ISC stateful resources as high-risk even on update.
- `/gh-bulk-issues` — create multiple GitHub issues from a template + item list. Preview-before-create. Enforces anonymization on public repos.
- `/dokploy-log-triage` — fetch Dokploy deployment logs, match against known failure signatures (OOM, IPv6 healthcheck, port conflict, missing env, cert), propose a concrete fix.

## Requirements

- `gh` CLI authenticated (for `/gh-bulk-issues`).
- `terraform` CLI (for `/terraform-plan-review`).
- **Dokploy MCP server** configured (for `/dokploy-log-triage`).

## Install

```
/plugin install workflow@trustin-ai
```

## Usage

> "Review le plan terraform avant apply" → `/terraform-plan-review`
>
> "Crée ces 4 issues" → `/gh-bulk-issues`
>
> "Le déploiement a échoué, aide-moi à débugger" → `/dokploy-log-triage`
