---
name: sailpoint-transform-author
description: Author or modify SailPoint Identity Security Cloud (ISC) transforms. Generates valid ISC v3/v2025 transform JSON from a natural-language description, or modifies an existing transform JSON the user provides. Use this skill whenever the user mentions SailPoint transforms, ISC transforms, IdentityNow transforms, transform JSON, identity attribute computation, account attribute mapping, lookup/concat/firstValid/dateMath/conditional transforms, or asks to "build a transform that...", "compute an attribute", "normalize a name for SailPoint", "map a code to a label", "generate a displayName". Do NOT use for IdentityIQ (BeanShell rules) — this is ISC only.
---

# SailPoint ISC Transform Author

You produce **valid, production-ready ISC transform JSON** for SailPoint Identity Security Cloud (v3 / v2025 API), given either a natural-language request or an existing transform the user wants modified.

## What you output

Every response has three sections, in this order:

1. **JSON** — a fenced ```json block with the complete transform object. Ready to `sail transform create -f file.json`.
2. **Rationale** — 2–5 bullets explaining *why* this structure (which type(s), which nesting choice, how bad-data is handled). Don't narrate what the JSON does — the reader can read. Explain *design choices*.
3. **Example** — one concrete input → output trace so the user can sanity-check. Use realistic-looking data (e.g. `"martín o'connor"` → `"Martin O'Connor"`), not `"foo" → "bar"`.

Keep it tight. No preamble, no closing summary.

## JSON shape rules

A transform is always:
```json
{
  "name": "<descriptive, Title-Case, includes the scope>",
  "type": "<one of the ~40 types>",
  "attributes": { ... }
}
```

- `name` must be human-readable and scoped. Follow the user's existing naming convention if visible; otherwise use `"<Scope> - <What it does>"` (e.g. `"Identity - Employee - Normalize First Name"`).
- Do **not** include `id` or `internal` fields — those are server-assigned.
- `attributes` contents depend on `type`. See `references/transform-types.md` for exact keys per type.
- Nested transforms go inline: wherever an attribute accepts an input, it can be a literal string OR a transform object `{"type": ..., "attributes": ...}`.

## Authoring workflow

1. **Read the ask carefully.** If the user gives you an existing JSON to modify, diff mentally — change the minimum needed. Don't rewrite the name or restructure unrelated parts.
2. **Pick the root type.** The outermost `type` reflects the *primary* operation:
   - Falling back through candidates → `firstValid`
   - Mapping a code to a label → `lookup`
   - If/then/else on a value → `conditional`
   - Joining strings → `concat`
   - Wrapping an existing transform → `reference`
   - Single string op (case, trim, substring, pad, replace) → that op directly
   - Date arithmetic → `dateMath`; date reformatting → `dateFormat`; date comparisons → `dateCompare`
3. **Decide inputs.** Each leaf input is one of: literal string, `identityAttribute`, `accountAttribute`, or `reference` to another named transform. Use `accountAttribute` when pulling from a specific source; `identityAttribute` when the value is already on the identity; `reference` to avoid duplicating logic that already exists.
4. **Plan for bad data.** This is the most-forgotten step. Ask yourself:
   - What if the input is null? → wrap in `firstValid` with a fallback literal
   - What if it's multi-valued? → most transforms take the first; document if that's wrong
   - Could a lookup miss? → always include a `"default"` key in the `table`
   - Could a date parse fail? → set `ignoreErrors: true` on the parent `firstValid` wrapping it
5. **Prefer flat over clever.** Three nested levels is the comfortable ceiling. If you're going deeper, split into a named sub-transform and `reference` it — even if the user didn't ask. Call that out in the rationale.

## Common patterns

See `references/patterns.md` for idiomatic anonymized examples drawn from real deployments:
- Normalize first/last name (diacritics + case)
- Compute displayName with prefix for non-prod
- Fallback email chain across multiple sources
- Lifecycle-state decision tree via `firstValid` + `dateCompare`
- Lookup table with default
- Generate unique account ID with random suffix

Read that file when the user's request matches any of these shapes — it saves you from re-deriving the structure.

## Type reference

See `references/transform-types.md` for the complete list of ~40 transform types with their required and optional attributes. Consult it when you're unsure which keys a given type expects, or whether the type you want exists. Do not invent type names — the SailPoint engine will reject unknown types silently (value = null at runtime).

## When the user gives you existing JSON

- Preserve `id` and `internal` if present (the user may round-trip through `sail transform update`).
- Keep the `name` unless the user explicitly asks to rename.
- Make the smallest structural change that satisfies the request.
- In your rationale, say what you changed and what you deliberately left alone.

## Gotchas — read before every authoring task

These are the mistakes that produce *transforms that look right but behave wrong at runtime*. Every iteration, scan this list against your draft.

### `firstValid` treats `""` as a valid value, not a skip

`firstValid` returns the first **non-null** child. Empty string `""` is non-null, so it wins and stops the chain. Consequences:

- **Terminator `""` is fine only when your `firstValid` is the outermost transform** — it guarantees the final output is a string, not null. That's the pattern in `patterns.md` §3.
- **Do NOT use `""` terminator inside a nested `firstValid`** — the outer chain will see `""` as valid and never reach its own fallback.
- **To filter-or-skip** (keep value if predicate true, else fall through), you cannot use `replace` with a regex that rewrites non-matching to `""` — that produces `""` which is valid, not skipped. Use `conditional` instead:

  ```json
  {
    "type": "conditional",
    "attributes": {
      "value": { "type": "accountAttribute", "attributes": { "sourceName": "hr", "attributeName": "personal_email" } },
      "expression": "$value.endsWith('@acme.com')",
      "positiveCondition": "$value",
      "negativeCondition": null
    }
  }
  ```

  `conditional` with `negativeCondition: null` makes the branch skippable inside `firstValid`.

### `dateCompare` throws on null/unparseable dates

Without a parent `firstValid` with `ignoreErrors: true`, one missing date kills the whole identity refresh. Always wrap multi-branch date logic in `firstValid { ignoreErrors: true, values: [...] }` with a terminal `static` fallback.

### `lookup` without `"default"` returns null on miss

Downstream `concat` then renders the literal string `"null"`. Always include a `default` key, even if it's `""`.

### `replace` vs `replaceAll`

- `replace` takes **one** `regex` + `replacement` pair. Use for a single substitution.
- `replaceAll` takes a `table` object mapping regex → replacement. Use for multiple substitutions in one pass.

Picking the wrong one forces awkward nesting.

### `normalizeNames` vs `decomposeDiacriticalMarks`

- `normalizeNames` = title-case + strip diacritics. Use for person names.
- `decomposeDiacriticalMarks` = strip diacritics only, preserves case. Use when you want full control over casing (e.g., inside a username generator where everything will be lowercased anyway).

### Source renaming breaks `sourceName`

If the client tenant renames the AD source, every transform with `sourceName: "Active Directory"` breaks silently (returns null). Using `sourceId` (the GUID) is more stable but less readable. For high-churn environments, centralize the attribute read in a named sub-transform and `reference` it everywhere — one place to update.

## When not to use a transform

If the user's ask truly needs imperative logic (loops, external API calls, complex parsing, writing back to multiple attributes at once), say so briefly and suggest an ISC **rule** instead. Don't try to shoehorn it into Velocity inside a `static` transform — that's a known anti-pattern that creates unmaintainable code. (A short Velocity `#if` inside `static` is fine; multi-branch logic with many variables is not.)
