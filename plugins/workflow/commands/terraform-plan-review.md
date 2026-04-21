---
description: Run terraform plan, parse the JSON output, and produce a risk-tiered summary flagging destroys/replaces and ambiguous changes before apply.
allowed-tools: Bash, Read
---

Review a Terraform plan with an emphasis on **what changes, why, and what's dangerous** — before the user decides to apply.

## When to use

- User says "review the plan", "check the terraform plan", "what does this plan do", or starts a session by asking for confirmation before apply.
- Before any `terraform apply` on an IaC repo (especially those managing stateful IAM / identity resources).

## Steps

1. Confirm the working directory is a Terraform root (`*.tf` + `.terraform/`).
2. Accept (in this order):
   - Existing plan file given by the user (`-plan=<path>`) — read it directly via `terraform show -json <path>`.
   - Fresh plan: run `terraform plan -out=tfplan.bin -no-color` then `terraform show -json tfplan.bin > tfplan.json`. Delete `tfplan.bin` afterwards; keep `tfplan.json` in a tmp location, not committed.
3. Parse `tfplan.json`:
   - Walk `resource_changes[]`, group by `change.actions` (`create`, `update`, `delete`, `delete+create` = replace, `no-op`, `read`).
   - For each action bucket, aggregate `(provider, type, count)`.
   - Collect the full address list for anything in `delete` or `delete+create`.
4. Score each change:
   - 🔴 **High risk**: `delete` or `delete+create` on stateful resources (databases, identity sources, access profiles, roles, secrets, S3 buckets, DNS records). Any `delete` where the resource type contains `source`, `role`, `access_profile`, `identity`, `db`, `database`, `secret`, `bucket`, `dns`, `record`, `certificate`.
   - 🟠 **Medium risk**: `delete+create` on non-stateful resources, `update` that modifies `force_new` attributes (detectable via `replace_paths`).
   - 🟡 **Low risk**: plain `update` touching only non-sensitive attributes.
   - 🟢 **Safe**: `create` on new resources, `read` data sources, `no-op`.
5. For SailPoint ISC IaC: flag any change to `sailpoint_source`, `sailpoint_access_profile`, `sailpoint_role`, `sailpoint_identity_profile`, `sailpoint_form` as 🔴 regardless of action — these have production impact even on update.

## Output format

```
# Terraform plan review

**Totals:** +<N> create · ~<N> update · -<N> destroy · ↻<N> replace · <N> no-op

## 🔴 High risk (<N>)
- sailpoint_source.adp — **DESTROY** — stateful, breaks aggregations
- sailpoint_role.finance — **REPLACE** — recreation drops entitlements

## 🟠 Medium risk (<N>)
- aws_lb_listener.https — replace (forces_new on certificate_arn)

## 🟡 Low risk (<N>)
- 12 × sailpoint_access_profile — tag updates only

## 🟢 Safe (<N>)
- 4 × create (new roles)

## Recommendation
- <one line: "safe to apply" / "review line X before apply" / "do NOT apply, confirm destroy list">
```

## Guardrails

- Never run `terraform apply` from this command. Review-only.
- Never print full JSON — summarise.
- If `terraform plan` fails (auth, state lock, missing vars), report the exact error and suggest the fix (e.g., `sail login --env <tenant>`, `terraform force-unlock <id>`).
- For SailPoint plans: never invoke `curl` or raw HTTP for any sanity check — use the `sail` CLI only.
