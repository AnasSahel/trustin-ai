---
name: sailpoint-transform-refactor
description: Refactor a SailPoint Identity Security Cloud (ISC) transform that has become hard to maintain — flatten excessive nesting, extract reusable sub-transforms via `reference`, eliminate duplication, replace anti-patterns. Use this skill whenever the user says their transform is "too complex", "deeply nested", "unreadable", "hard to maintain", "needs to be split up", "should be DRY'd", "uses copy-pasted logic across transforms", or asks to "clean up" / "simplify" / "modernize" an ISC transform. Also use when reviewing legacy transforms inherited from a previous integrator.
---

# SailPoint ISC Transform Refactorer

You take an existing ISC transform that works but is painful to maintain, and produce a cleaner, more modular version. The behavior must remain identical — refactoring changes structure, not semantics.

## What you output

Three sections, in this order:

1. **Diagnosis** — what's wrong with the current shape (2–4 bullets). Be specific: cite nesting depth, duplicated subtrees, anti-patterns. Don't generically say "it's complex".
2. **Refactored JSON** — one or more transforms in fenced ```json blocks. If you extracted sub-transforms, show each one separately, in dependency order (innermost first), with the root last. Each gets a short header (`### Sub-transform: <name>`).
3. **What changed and why** — 3–5 bullets mapping each structural change to the maintainability problem it solves. Also flag what you deliberately left alone.

If behavior would change, **stop** and say so explicitly. Ask the user to confirm the behavior change before producing JSON. Silent semantic drift during refactor is the worst possible outcome.

## When to refactor

Trigger refactoring when you see any of:

- **Nesting depth > 3** — hard to audit at a glance
- **Same subtree repeated ≥ 2 times** in one transform — DRY violation, drift risk
- **Same subtree repeated across multiple transforms** — extract once, `reference` everywhere
- **Velocity multi-branch `#if` in `static.value`** — replace with nested `firstValid`/`conditional`/`dateCompare`
- **`replace` used as a filter** (rewrites non-matching to `""`) — wrong semantics inside `firstValid`; replace with `conditional { negativeCondition: null }`
- **Hardcoded `sourceId` GUIDs** copied everywhere — extract `accountAttribute` reads into named transforms
- **`firstValid` with no `ignoreErrors`** wrapping date logic — add the flag
- **`lookup` with no `"default"` key** — add `"default"` (even if empty)
- **Long `concat` chain** with literal separators — replace with `join` + `separator`

## Refactoring techniques

### 1. Extract a sub-transform via `reference`

When the same subtree appears more than once, lift it into its own named transform and replace each occurrence with a `reference`. Naming convention:
- `<Scope> - <Source> - Get <Attribute>` for attribute-extraction reads
- `<Scope> - Compute <Result>` for derived values
- `<Scope> - Lookup <Domain>` for tables

Example: replace 4 inline `accountAttribute` reads of `hr-workday.end_date` with a single transform `Account - HR - Get End Date` and four `reference` calls.

### 2. Flatten excessive nesting

Three levels deep is the comfortable ceiling. Beyond that, extract intermediate steps into named sub-transforms. The result has more transforms but each is easier to read and to test in isolation via `sail transform preview`.

### 3. Replace Velocity logic with native types

Velocity in `static.value` is opaque, ungreppable, and hard to debug. A multi-branch `#if` chain becomes a nested `firstValid` with `conditional` or `dateCompare` children. Single-line Velocity (one expression, no branching) is fine — don't over-refactor.

### 4. Replace `replace`-as-filter with `conditional`

When `replace` rewrites a non-matching value to `""` and lives inside `firstValid`, the empty string is treated as valid (bug). Swap for:
```json
{
  "type": "conditional",
  "attributes": {
    "value": <input>,
    "expression": "<predicate>",
    "positiveCondition": "$value",
    "negativeCondition": null
  }
}
```

### 5. Use `join` instead of `concat` for separator-heavy strings

`concat` with literal separators interleaved (`["a", " - ", "b", " - ", "c"]`) is noisier than `join` with a `separator` attribute. For 2 elements concat is fine; from 3+, prefer `join`.

## Behavior preservation — non-negotiable

Before producing the refactored JSON, mentally simulate both versions on these inputs:
- A normal happy-path input
- A null input on every leaf attribute (one at a time)
- An empty-string input on each
- A multi-valued input (where applicable)

If outputs differ at any point, you've changed semantics. Either revert or escalate to the user with the explicit diff. Refactoring that "fixes" a bug along the way is not a refactor — it's a behavior change wearing a refactor's clothes. Disclose it.

## Reference

- `references/transform-types.md` for type schemas — verify your refactor uses valid types and required attributes.
- `references/patterns.md` for the idiomatic target shapes.

## When NOT to refactor

- The transform is already at idiomatic complexity for what it does (e.g., a 6-branch lifecycle decision tree is unavoidably ~5 levels deep).
- The user asked for a fix to a bug — use the debug skill, not this one.
- The user asked for a behavior change — use the author skill.
- The transform is owned by an external system that expects this exact JSON shape.
