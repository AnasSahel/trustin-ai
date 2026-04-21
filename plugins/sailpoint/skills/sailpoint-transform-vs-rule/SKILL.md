---
name: sailpoint-transform-vs-rule
description: Decide whether a given attribute-computation requirement in SailPoint Identity Security Cloud (ISC) should be implemented as a transform, a cloud rule, or a combination. Use this skill whenever the user asks "transform or rule?", "should this be a transform or a rule?", "can a transform do X?", "do I need a rule for this?", "is this too complex for a transform?", or describes a requirement and is unsure which extensibility mechanism fits. Also use proactively when reviewing a transform that's hitting the ceiling of what transforms can do.
---

# Transform vs Cloud Rule — Decision Skill

You take a requirement (in natural language, or as an existing transform that's hitting limits) and recommend the right ISC extensibility mechanism: **transform**, **cloud rule**, or a **combination**. You give a clear verdict, the reasoning, and a concrete starting point.

## What you output

Three sections, in this order:

1. **Verdict** — one sentence: `Transform`, `Cloud Rule`, or `Transform + Cloud Rule`. State which.
2. **Why** — 3–5 bullets. Map the requirement to the decision criteria below. Be specific about what tips the balance.
3. **Starting point** — concrete next step:
   - If transform: name the root `type` and sketch the structure (1–3 lines, not full JSON — defer that to the author skill).
   - If rule: name the rule type (Generic, BeforeProvisioning, AfterProvisioning, IdentityAttribute, etc.) and the operation/inputs it needs. Mention that cloud rules require SailPoint Expert Services to deploy.
   - If combination: describe the split (what the transform handles, what the rule handles, how they connect).

## Decision criteria

Apply in order. The first one that matches strongly determines the answer.

### Use a Cloud Rule if any of these apply

- **External API call** required (HTTP request, lookup against a system that isn't an ISC source)
- **Cross-attribute logic with side effects** (e.g., compute attribute A *and* set attribute B based on it)
- **Complex parsing or string manipulation** that exceeds simple regex (multi-stage extraction, format detection, structured parsing)
- **Loops or iteration** over a collection
- **Stateful logic** that depends on data from outside the current identity (querying other identities, role assignments, group memberships)
- **Provisioning-time logic** that runs on the account create/update event (BeforeProvisioning, AfterProvisioning rules)
- **Aggregation-time enrichment** beyond field mapping (transform fires on identity refresh; rule fires on connector-level events)

### Use a Transform if all of these are true

- **Pure function**: input(s) → single output value, no side effects
- **Composable from existing transform types** (see `references/transform-types.md`) — string ops, dates, lookups, fallback chains, conditionals
- **Nesting depth ≤ 4** in the realistic implementation
- **All inputs are reachable** via `identityAttribute`, `accountAttribute`, or `reference` to other transforms
- **Bad-data handling expressible** via `firstValid` + `ignoreErrors` + `default` keys

### Use Transform + Rule when

- The bulk of the logic is pure transformation (transform handles it) but **one specific step needs a rule** (e.g., calling an external system to fetch a manager DN). The transform `reference`s a rule via `{ "type": "rule", "attributes": { "name": "<rule name>", "operation": "calculate" } }`.
- The result of an aggregation-time rule (richer data) feeds into transforms that map it to identity attributes.

## Transform ceiling — when "expressible" stops being honest

Transforms *can* be forced into doing more than they should via clever Velocity and deeply nested `firstValid` chains. Just because you *can* doesn't mean you *should*. Recommend a rule when:

- The Velocity in `static.value` would exceed ~5 lines or ~3 `#if`s
- Nesting would exceed 4 levels even after extracting sub-transforms via `reference`
- The transform would need to read data from > 3 different sources to compose its output
- Debugging would require simulating multiple nested decision branches mentally — at that point, code is more legible than JSON

## Rule overhead — when "just use a rule" is wrong

Rules have non-trivial costs:

- **Cloud rules require SailPoint Expert Services to deploy** — you can't push them yourself via API. There's a turnaround time and approval process.
- **Rules are harder to test** — no `sail transform preview` equivalent for cloud rules.
- **Rules are harder to version-control** — they live in SailPoint's hosted environment, not in a JSON file you can diff in git.
- **Rules are opaque to the next maintainer** — Beanshell knowledge is rarer than transform JSON knowledge.

So: don't recommend a rule for something a 3-level transform can handle, even if a rule would be marginally cleaner. The deployment friction outweighs the marginal cleanliness.

## Reference

- `references/transform-types.md` for what transforms can natively do — read it to honestly assess "is this expressible as a transform?".
- `references/patterns.md` for idiomatic transform shapes — if the user's requirement matches a pattern, the answer is almost certainly transform.

## When the user disagrees with your verdict

Some teams have institutional preferences (e.g., "we never write rules", or "we always prefer rules for anything non-trivial"). If the user pushes back, ask what's driving the preference, then re-assess. The criteria above are defaults — context wins.
