# sailpoint

SailPoint ISC toolkit for Claude Code — transforms, reporting, and Terraform provider maintenance.

## Contents

### Skills (transforms)

Invoke by describing the task; Claude Code routes based on the skill descriptions.

- `sailpoint-transform-author` — write a new transform from a plain-language spec
- `sailpoint-transform-explain` — explain what a given transform does, step by step
- `sailpoint-transform-debug` — diagnose why a transform returns null / wrong value
- `sailpoint-transform-refactor` — simplify / extract shared logic across transforms
- `sailpoint-transform-vs-rule` — decide whether a need should be a transform or a cloud rule, with tradeoffs

Each skill also has a `-workspace` variant that works on a pre-assembled workspace directory.

### Agents

- `sailpoint-tf-drift-analyzer` — analyse drift between a Terraform resource and the live tenant state
- `sailpoint-workflow-test-runner` — trigger a workflow via `/test` with a sample payload and return a step-by-step trace
- `terraform-provider-sailpoint-maintainer` — maintain the OSS `AnasSahel/terraform-provider-sailpoint-isc-community` provider: scan for resource gaps vs the upstream v2025 API, scaffold resource + datasource + acceptance tests, open anonymized PRs

### Shared references

`_shared/references/` — transform patterns and type catalog, consumed by the skills above.

## Requirements

- **`sail` CLI** configured with the target tenant environment (`--env <name>`). All SailPoint API calls go through `sail` — never `curl`.
- For the Terraform provider agent: `gh` CLI authenticated, Go ≥ 1.22, repo cloned locally.

## Install

```
/plugin install sailpoint@trustin-ai
```

## Usage

> "Write a transform that extracts the email domain" → `sailpoint-transform-author`
>
> "Why does this transform return null?" → `sailpoint-transform-debug`
>
> "Check drift on this sailpoint_source resource" → `sailpoint-tf-drift-analyzer`
>
> "Which SailPoint resources are missing from the community provider?" → `terraform-provider-sailpoint-maintainer --scan`

## Safety notes

- All SailPoint API work is **read-only** except what's strictly required (e.g., the provider agent may write under a test tenant via acceptance tests).
- The Terraform provider repo is **public** — the agent enforces anonymization (no tenant IDs, DNs, client names) on every issue / PR / example it produces.
- Reporting agents only write inside the vault, never to SailPoint.
