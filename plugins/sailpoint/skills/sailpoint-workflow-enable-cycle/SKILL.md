---
name: sailpoint-workflow-enable-cycle
description: Safe disable-apply-reenable cycle for SailPoint ISC workflows. The ISC API refuses to update a workflow while it's `enabled=true` (error "failed to update workflow because it is enabled: not allowed"). This skill wraps the disable → apply → verify → re-enable sequence with safety nets (idempotent disable, verify zero-diff post-apply, restore enabled state only if it was previously true, never leave a workflow accidentally disabled). Use before any `tofu apply` that touches `sailpoint_workflow.*` or its trigger, or before `sail api patch` updates on workflow definitions.
---

# SailPoint Workflow Enable-Cycle Skill

The ISC API has a known quirk: workflows with `enabled=true` reject any update to their definition or trigger with a `400 Bad Request` — `"failed to update workflow because it is enabled: not allowed"`. This skill codifies the safe workaround.

Related provider issue: [AnasSahel/terraform-provider-sailpoint-isc-community#102](https://github.com/AnasSahel/terraform-provider-sailpoint-isc-community/issues/102).

## When to use

- Before `tofu apply` on `sailpoint_workflow.*` or `sailpoint_workflow_trigger.*`
- Before a `sail api patch`/`put` on a workflow definition
- When you see the error `workflow is enabled but executed as a test workflow` from the `/test` endpoint (the test endpoint also requires `enabled=false`)

## When NOT to use

- If the workflow is already `enabled=false` (no cycle needed — just apply)
- For a brand new workflow being created by TF (no current state, no disable needed)
- For a workflow you want to leave disabled (skip the re-enable step)

## Required inputs

- `workflow_id` — UUID of the workflow
- `env` — `sail` CLI env
- `apply_cmd` — the actual TF or API command to run (this skill surrounds it, doesn't compose it)
- `preserve_enabled_state` (optional, default `true`) — if the workflow was `enabled=true` before, re-enable after the apply

## The safe cycle

### Step 1 — Read initial state

```bash
was_enabled=$(sail api get "/v2025/workflows/<workflow_id>" --env <env> 2>&1 \
  | grep -v '^[0-9]\{4\}/' | grep -v '^Status:' \
  | tail -1 \
  | grep -oE '"enabled":(true|false)' | cut -d':' -f2)

echo "Initial enabled state: $was_enabled"
```

If `was_enabled=false`, skip to Step 3 (no disable needed). Otherwise continue.

### Step 2 — Disable

```bash
sail api patch "/v2025/workflows/<workflow_id>" \
  --body '[{"op":"replace","path":"/enabled","value":false}]' \
  --env <env>
```

Verify response contains `"enabled":false`. If anything else, stop and report — do not proceed to apply.

### Step 3 — Execute the provided command

This is the user's own `apply_cmd`. Common forms:

```bash
# Terraform targeted apply
make apply ENV=<env> TARGET=sailpoint_workflow.<name>

# Untargeted
make apply ENV=<env>

# Direct API update
sail api put "/v2025/workflows/<workflow_id>" --body '<json>' --env <env>
```

Capture exit code. If non-zero, **do NOT** re-enable blindly — report the failure and let the user decide (the workflow may be in a broken state; re-enabling would run broken code on real events).

### Step 4 — Verify

For Terraform:

```bash
make plan ENV=<env> TARGET=sailpoint_workflow.<name>
# Expected: "No changes." — confirms apply was idempotent
```

For direct API: fetch the workflow again and compare key attributes to expected.

If plan shows non-zero drift after apply, investigate before proceeding.

### Step 5 — Re-enable (conditional)

If `was_enabled=true` AND `preserve_enabled_state=true` AND the apply/verify succeeded:

```bash
sail api patch "/v2025/workflows/<workflow_id>" \
  --body '[{"op":"replace","path":"/enabled","value":true}]' \
  --env <env>
```

Verify response contains `"enabled":true`. Final state report.

### Step 6 — Final state report

Produce a one-line summary:

```
✅ Workflow <id> cycled: was_enabled=<bool>, apply <OK|FAILED>, now_enabled=<bool>
```

## Safety rules

- **Never** re-enable a workflow if the apply failed. The definition may be in a broken state, and re-enabling sends broken logic to production events.
- **Never** assume the initial state — always read it explicitly. A workflow may have been disabled manually and left that way intentionally.
- **Never** skip Step 4 (verification). Otherwise you risk re-enabling a workflow that didn't actually get updated (if the apply silently failed).
- If an interrupt/error happens between Step 2 and Step 5, **always** attempt a restore of the initial `enabled` state before exiting. Leaving a production workflow disabled without notice is an outage.

## Example — full script

```bash
#!/usr/bin/env bash
set -euo pipefail

WF_ID="$1"
ENV="$2"
shift 2
APPLY_CMD=("$@")

cleanup() {
  if [ "$was_enabled" = "true" ] && [ "${enabled_now:-false}" = "false" ]; then
    echo "⚠ Restoring enabled=true on exit"
    sail api patch "/v2025/workflows/$WF_ID" \
      --body '[{"op":"replace","path":"/enabled","value":true}]' --env "$ENV" >/dev/null
  fi
}
trap cleanup EXIT

# Step 1
was_enabled=$(sail api get "/v2025/workflows/$WF_ID" --env "$ENV" 2>&1 \
  | grep -v '^[0-9]\{4\}/' | grep -v '^Status:' | tail -1 \
  | grep -oE '"enabled":(true|false)' | cut -d':' -f2)
echo "Initial: enabled=$was_enabled"

# Step 2 (if needed)
if [ "$was_enabled" = "true" ]; then
  sail api patch "/v2025/workflows/$WF_ID" \
    --body '[{"op":"replace","path":"/enabled","value":false}]' --env "$ENV" >/dev/null
  echo "Disabled"
fi

# Step 3
"${APPLY_CMD[@]}"

# Step 5
if [ "$was_enabled" = "true" ]; then
  sail api patch "/v2025/workflows/$WF_ID" \
    --body '[{"op":"replace","path":"/enabled","value":true}]' --env "$ENV" >/dev/null
  enabled_now="true"
  echo "Re-enabled"
fi

echo "✅ Cycle complete"
```

## Output of this skill

When invoked, produce:

1. **Plan** — the sequence of commands that will be run, with the initial `was_enabled` state
2. **Confirmation ask** — "Proceed with disable → apply → re-enable?"
3. **After execution** — the final state report + verify output

## Related

- Provider issue: https://github.com/AnasSahel/terraform-provider-sailpoint-isc-community/issues/102
- `sailpoint-workflow-test-runner` agent uses the same pattern internally for `/test` endpoint calls
- ISC official docs on workflows: https://developer.sailpoint.com/docs/api/v2025/workflows
