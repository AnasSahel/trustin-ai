---
name: sailpoint-transform-adopt
description: Adopt an existing SailPoint ISC transform under Terraform management. Fetches the transform JSON via sail CLI, generates a new `sailpoint_transform` resource with a name conforming to the `<ClientPrefix> - <Scope> - <Action>` convention (or your project's convention), lists the Terraform consumers that need to be repointed, and tracks the old transform as an orphan to delete post-validation. Use when you have a transform living in the tenant (legacy name like `id-legacy-*`, UI-created, etc.) that you want to bring under TF with a proper name. This skill only produces the TF resource and a changeset plan — it does NOT write to ISC or apply Terraform.
---

# SailPoint Transform Adoption Skill

Bringing an existing tenant transform under Terraform management, with rename to a conforming convention, in a safe non-destructive way.

## When to use

- User mentions a legacy transform name (e.g. `id-legacy-password-generator`, `id-oldcontractor-get-*`, any non-`<ClientPrefix> - <Scope> - <Action>` name)
- User wants to "terraformize" or "adopt" a transform that exists in the tenant
- User wants to rename a transform but preserve runtime behavior
- User has duplicated UI-created transforms that should be consolidated under TF

## When NOT to use

- If the transform doesn't exist yet — use standard `sailpoint-transform-author` skill instead
- If the transform is already TF-managed — just rename it in-place via Terraform (state rename or create-new-then-state-rm)
- If the goal is to delete the transform — use a state-rm + API delete flow, not adoption

## Required inputs

- `transform_name` or `transform_id` — identifier of the legacy transform
- `env` — `sail` CLI env (e.g. `<tenant>-sb`, `<tenant>`)
- `target_scope` — the scope for the new name (`AD`, `Contractor`, `Shared`, `Employee`, `External`, `IDN`, ...)
- `target_action` — the action/purpose (e.g. `Generate Password`, `Lookup City`, `Compute LCS`)
- `tf_file` — target file for the new resource (defaults based on scope: `transforms-shared.tf`, `transforms-contractor.tf`, etc.)

If any are missing, ask. Don't guess the scope — that decision drives the final name + file location.

## Output

Four deliverables, in this order:

### 1. Diagnostic — current transform JSON

Fetch via :

```bash
sail api get '/v2025/transforms?filters=name%20eq%20%22<legacy_name>%22' --env <env> 2>&1 \
  | grep -v '^[0-9]\{4\}/' | grep -v '^Status:' \
  | tail -1
```

(Or by id: `sail api get "/v2025/transforms/<id>" --env <env>`)

Present the fetched JSON in a fenced block so the user can verify before adoption. Note the `type`, `id`, `internal` flag, and whether the content uses cloud rules / lookups / nested refs.

### 2. TF resource — ready to paste

Generate the `sailpoint_transform` HCL block following the project's Terraform style guide (`.claude/rules/terraform-style.md` in the target repo if it exists). Key rules:

- **New name** : `<ClientPrefix> - <target_scope> - <target_action>` (or client's equivalent convention)
- **HCL resource name** : `snake_case`, ex: `shared_ad_generate_password` for scope=AD, action=Generate Password
- **Always** use `core::jsonencode()` (namespaced form) for the `attributes` field
- **Always** use HCL-native keys inside the jsonencode (bareword, `=` aligned), never raw JSON strings
- **Preserve** the transform's semantic JSON verbatim — only the wrapping resource + name changes
- **Header comment** explaining (a) what the transform does, (b) that it replaces the legacy `<legacy_name>`, (c) that the legacy one is left orphaned and needs manual cleanup

Example skeleton:

```hcl
# Transform <Scope> - <Action>
#
# <one-sentence description of behavior>
# <one-sentence usage: who consumes this transform>
#
# Remplace le legacy `<legacy_name>` (meme JSON, nom non conforme a la
# convention `<ClientPrefix> - <Scope> - <Action>`). L'ancien reste orphelin cote
# tenant (pas TF-managed, pas supprime pour eviter de casser un eventuel
# consumer non-TF). A nettoyer manuellement apres verification.
resource "sailpoint_transform" "<hcl_name>" {
  name = "<ClientPrefix> - <Scope> - <Action>"
  type = "<type_from_json>"
  attributes = core::jsonencode({
    # <transposed verbatim from legacy JSON, bareword keys, aligned =>
  })
}
```

### 3. Consumers to repoint

Grep the TF codebase for references to the legacy name or id:

```bash
grep -rn "<legacy_name>\|<legacy_id>" --include="*.tf" <tf_dir>
```

For each hit, produce a patch suggestion changing the reference from the legacy literal to `sailpoint_transform.<hcl_name>.name`. Reference style examples:

```hcl
# Before
transform = core::jsonencode({
  type = "reference"
  attributes = { id = "id-legacy-password-generator" }
})

# After
transform = core::jsonencode({
  type = "reference"
  attributes = { id = sailpoint_transform.shared_ad_generate_password.name }
})
```

**Warning** : if the legacy transform is referenced inside a string-encoded JSON (e.g. a `core::jsonencode({...})` wrapping for a provisioning policy's `transform` attribute), the reference is a **string literal**, not an HCL interpolation. The interpolation happens at jsonencode time — so `id = sailpoint_transform.x.name` inside the jsonencode does work. Verify the context.

### 4. Orphan cleanup tracking

Generate a one-paragraph "TODO" block for the project's `TODO.md` (or equivalent backlog):

```markdown
## Transform orphelin à nettoyer

`<legacy_name>` remplacé par `<ClientPrefix> - <Scope> - <Action>` (même JSON, nom conforme convention). L'ancien reste côté tenant non-TF-managed.

IDs à supprimer :
- <env1> : `<id_env1>`
- <env2> : `<id_env2>` (si multi-env)

Prérequis avant suppression : vérifier qu'aucun consumer non-TF (workflows UI-made, campaigns, reports) ne référence encore le nom legacy.

Procédure :
\`\`\`
sail api delete /v2025/transforms/<id> --env <env>
\`\`\`
```

Also suggest creating a tracking issue if the project uses an issue tracker (GitHub, Linear, or a vault-based ticketing scheme like `PROJ-NNN`). The issue should include: both env IDs, verification checklist, deletion procedure, rollback (no rollback needed — re-create via `sail api post` with the same JSON).

## Step-by-step procedure

1. **Confirm scope/action** — ask if not provided. Validate the name lint: `<ClientPrefix> - <Scope> - <Action>` (Title Case, singulier, action en anglais).
2. **Fetch JSON** via `sail api`. If multiple envs have the transform, fetch both and verify the JSON matches. If they diverge, flag it and ask the user which is source of truth.
3. **Draft TF resource** with the conventions above. Align `=` signs.
4. **Grep consumers** in the TF dir. List each.
5. **Produce the orphan TODO** with IDs from all envs.
6. **Output** : `sail api` response excerpt + proposed HCL + consumer patch list + TODO snippet. Do NOT write files unless the user explicitly asks you to patch the TF for them.

## Edge cases

- **Transform consumed by identity profile mappings** : the consumer list may include `identity-profile-*.tf` files. The reference there is typically `id = sailpoint_transform.x.name` inside a jsonencode for a `transform_definition`. Same pattern applies.
- **Transform consumed by a cloud rule** : cloud rules aren't TF-managed. Flag that the legacy name may still be in the cloud rule body. Grep isn't exhaustive for this — warn the user.
- **Transform of type `rule`** pointing to a SailPoint-internal rule (e.g. `"Cloud Services Deployment Utility"`) : this is a reference to a generic cloud rule, not something to adopt. Preserve the rule name verbatim in the JSON.
- **Multiple transforms with the same legacy name across envs** : some tenants have `id-legacy-password-generator` in both sandbox and prod with the same JSON but different ids. Only the name matters for the adoption — both envs' consumers will resolve to the same new name.

## Verification after user applies

Once the user has pasted the resource, updated consumers, and applied in sandbox:

```bash
# Verify the new transform exists
sail api get '/v2025/transforms?filters=name%20eq%20%22<ClientPrefix>%20-%20<Scope>%20-%20<Action>%22' --env <env>

# Verify the old one is orphaned (no more TF consumers)
grep -rn "<legacy_name>\|<legacy_id>" --include="*.tf" <tf_dir>
# Expected: zero hits after patches applied

# Verify zero drift
cd <tf_dir> && make plan ENV=<env>
# Or: tofu plan -var-file=environments/<env>.tfvars
```

Only then suggest proceeding to delete the legacy transform (guarded by the TODO + manual check of non-TF consumers).

## Why this skill exists

Rename-in-place of tenant transforms is risky because :
- The transform `id` in ISC references is actually the transform **name** (historical API quirk — the `id` field accepts either the UUID or the name)
- Non-TF consumers (cloud rules, UI workflows, campaigns, reports) may reference the legacy name and break silently on rename
- TF state hidden dependencies (transforms referenced inside jsonencoded policy fields) are easy to miss in a plain rename

Create-new + orphan-old + verify + delete is the safe pattern. This skill codifies it.
