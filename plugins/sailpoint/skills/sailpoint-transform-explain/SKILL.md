---
name: sailpoint-transform-explain
description: Explain a SailPoint Identity Security Cloud (ISC) transform JSON in plain language, including a flow diagram of how data moves through nested operations. Use this skill whenever the user pastes ISC transform JSON and asks "what does this do", "explain this transform", "I inherited this transform and don't understand it", "trace how this works", or shows a deeply-nested transform they want to understand. Also use when the user asks to document a transform, write a comment for it, or onboard someone new to it. Do NOT use to write or modify transforms (use sailpoint-transform-author for that).
---

# SailPoint ISC Transform Explainer

You take an ISC transform JSON (often deeply nested, often inherited from someone else) and produce a clear explanation of what it does, why it's structured that way, and how data flows through it.

## What you output

Three sections, in this order:

1. **TL;DR** — one sentence. What does this transform produce, given some input? Example: *"Returns the employee's email address, falling back across AD, HR, and personal email sources, with `@acme.com` filtering on the personal one."*
2. **Flow** — an ASCII tree showing the data path from inputs to output. Use indentation + arrows. Annotate each node with its `type`. Show concrete attribute names (`hr-workday.family_name`, not just "an account attribute").
3. **Walkthrough** — numbered list, one entry per non-trivial node, explaining *what it does* and *why it's there* (best guess at intent). Call out:
   - Bad-data handling (`ignoreErrors`, fallbacks, defaults)
   - Anti-patterns or fragility (see Gotchas)
   - Anything the next maintainer needs to know

Keep it readable. No wall of text. Use bullet points and short sentences.

## Flow diagram conventions

```
[ROOT type:firstValid]
├── [type:accountAttribute] ← Active Directory.mail
├── [type:accountAttribute] ← hr-workday.email
└── [type:replace] regex=^(?!.*@acme\.com$).*$ → ""
    └── [type:accountAttribute] ← hr-workday.personal_email
=> output
```

- `[type:X]` for each node
- `←` for inputs (account/identity attributes, references)
- `→` for transformations applied
- Indentation = nesting
- `=> output` at the bottom for the final result

For very large transforms (>15 nodes), you may collapse uninteresting subtrees as `[...subtree omitted, see node N below...]` and explain them separately in the walkthrough.

## How to read a transform

1. **Start at the root.** The root `type` tells you the *primary* operation. See `references/transform-types.md` for what each type does.
2. **Trace inputs depth-first.** Each `attributes.input` (or each entry in `attributes.values`) is a child node. Recurse into it before moving to siblings.
3. **Resolve `reference` nodes.** A `reference` calls another named transform (`attributes.id`). Note it in the flow as `[type:reference] → "<name>"` and explain that the logic continues in that other transform. Don't try to fetch and inline it unless the user asks.
4. **Identify the bad-data strategy.** Look for `firstValid`, `ignoreErrors`, terminal `static` fallbacks, `default` keys in `lookup` tables. Call out missing protections.
5. **Notice Velocity in `static`.** If the root is `static` with a `value` containing `$variables` and `#if`, the transform is executing template logic — explain the template, list the variable bindings, and trace what each one resolves to.

See `references/transform-types.md` for the per-type attribute schemas and `references/patterns.md` for idiomatic shapes you'll likely recognize.

## Gotchas to flag in your walkthrough

If the transform exhibits any of these, mention them — they're common bug sources:

- **`firstValid` with `""` terminator nested inside another `firstValid`** → outer chain never reaches its fallback (the inner `""` wins).
- **`lookup` without `"default"` key** → unmapped inputs return null, downstream concat shows `"null"` literal.
- **`dateCompare` not wrapped in `firstValid { ignoreErrors: true }`** → null/unparseable date kills the identity refresh.
- **`replace` used as a filter** (regex rewrites non-matching to `""`) → produces `""` which `firstValid` treats as valid; should be `conditional` with `negativeCondition: null`.
- **Hardcoded `sourceId` GUIDs** copied across many transforms → renaming/recreating the source breaks all of them silently.
- **Velocity multi-branch `#if` chains in `static.value`** → unmaintainable; should be split into nested `firstValid`/`conditional`.
- **Nesting depth > 4** → hard to audit; should be split into named sub-transforms via `reference`.

When you flag a gotcha, suggest the fix briefly (one sentence). Don't write the corrected JSON unless asked — that's the author skill's job.

## When the transform calls a `reference` you don't have

Say so explicitly: *"This transform delegates part of its logic to `<reference name>` — I can only explain what's visible here. Paste that transform too if you need the full picture."*

Don't speculate about what the referenced transform does. The whole point of `reference` is reuse, and guessing wrong sends the maintainer down a wrong trail.
