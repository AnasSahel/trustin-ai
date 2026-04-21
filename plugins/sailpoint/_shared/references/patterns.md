# Idiomatic ISC Transform Patterns

Anonymized patterns drawn from real production deployments. Use these as templates when the user's request matches one of these shapes — the structure is battle-tested.

Fake organization in examples: **Acme**, with HR source **hr-source-csv** and AD source **Active Directory**.

---

## 1. Normalize a name (title-case + strip diacritics)

Simple and preferred over hand-rolling `replaceAll` tables:

```json
{
  "name": "Identity - Employee - Normalize First Name",
  "type": "normalizeNames",
  "attributes": {}
}
```

The `normalizeNames` type handles title-casing and diacritic removal in one go. Input flows in automatically from the attribute mapping.

---

## 2. Concat with a prefix (e.g. non-prod displayName marker)

```json
{
  "name": "Identity - Employee - Compute DisplayName (Sandbox)",
  "type": "concat",
  "attributes": {
    "values": [
      "(SBX) ",
      { "type": "displayName", "attributes": { "input": "input" } }
    ]
  }
}
```

Using `displayName` as the inner transform delegates formatting to SailPoint's built-in.

---

## 3. Fallback email across multiple sources

```json
{
  "name": "Identity - Employee - Get Valid Email",
  "type": "firstValid",
  "attributes": {
    "values": [
      { "type": "accountAttribute", "attributes": { "attributeName": "mail", "sourceName": "Active Directory" } },
      { "type": "accountAttribute", "attributes": { "attributeName": "work_email", "sourceName": "hr-source-csv" } },
      ""
    ]
  }
}
```

- Order = priority.
- Empty string terminator means "if all else fails, empty, don't leave null" — avoids downstream breakage.
- ⚠️ The `""` terminator is safe **only because this `firstValid` is the outermost transform**. If you nest this inside another `firstValid`, the outer one will see `""` as valid and never reach its own fallback. See SKILL.md → Gotchas.

---

## 4. Lookup table with default

```json
{
  "name": "Identity - Lookup Company Name",
  "type": "lookup",
  "attributes": {
    "table": {
      "ACM": "Acme Corp",
      "ACMS": "Acme Solutions",
      "default": "N/A"
    }
  }
}
```

Always include `"default"`. Without it, unmapped keys return null and break concat/downstream transforms.

---

## 5. Lifecycle state decision tree (dateCompare inside firstValid)

```json
{
  "name": "Identity - Contractor - Compute LCS",
  "type": "firstValid",
  "attributes": {
    "ignoreErrors": true,
    "values": [
      {
        "type": "dateCompare",
        "attributes": {
          "firstDate": "now",
          "operator": "GT",
          "positiveCondition": "archived",
          "secondDate": {
            "type": "dateMath",
            "attributes": {
              "expression": "+60d",
              "input": { "type": "accountAttribute", "attributes": { "attributeName": "endDate", "sourceName": "hr-source-csv" } }
            }
          }
        }
      },
      {
        "type": "dateCompare",
        "attributes": {
          "firstDate": "now",
          "operator": "GT",
          "positiveCondition": "terminated",
          "secondDate": { "type": "accountAttribute", "attributes": { "attributeName": "endDate", "sourceName": "hr-source-csv" } }
        }
      },
      {
        "type": "dateCompare",
        "attributes": {
          "firstDate": "now",
          "operator": "GTE",
          "positiveCondition": "active",
          "secondDate": { "type": "accountAttribute", "attributes": { "attributeName": "startDate", "sourceName": "hr-source-csv" } }
        }
      },
      { "type": "static", "attributes": { "value": "scheduled" } }
    ]
  }
}
```

- `firstValid` scans children; each `dateCompare` returns its `positiveCondition` only if the comparison is true, else returns null and the chain continues.
- Terminal `static` guarantees a value.
- `ignoreErrors: true` essential because missing dates throw otherwise.

---

## 6. Named sub-transform invocation via `reference`

```json
{
  "name": "Identity - Employee - Get Country Label",
  "type": "reference",
  "attributes": { "id": "Identity - Lookup Country ISO to Name" }
}
```

Use this when the same logic is used in ≥2 places — author the inner transform once, reference it everywhere.

---

## 7. Generate account ID with random suffix (account create policy)

```json
{
  "name": "Account - Generate Unique ID",
  "type": "concat",
  "attributes": {
    "values": [
      {
        "type": "lower",
        "attributes": {
          "input": {
            "type": "substring",
            "attributes": {
              "input": { "type": "identityAttribute", "attributes": { "name": "firstname" } },
              "begin": 0, "end": 1
            }
          }
        }
      },
      {
        "type": "lower",
        "attributes": {
          "input": { "type": "identityAttribute", "attributes": { "name": "lastname" } }
        }
      },
      { "type": "randomNumeric", "attributes": { "length": 3 } }
    ]
  }
}
```

Pattern: first initial + last name + 3-digit random. `usernameGenerator` would be cleaner if you need uniqueness guarantees across existing accounts — use that when collisions matter.

---

## 8. Date reformat (HR-source `ISO8601` → `yyyy-MM-dd`)

```json
{
  "name": "Identity - Format HR Date",
  "type": "dateFormat",
  "attributes": {
    "inputFormat": "ISO8601",
    "outputFormat": "yyyy-MM-dd"
  }
}
```

---

## Anti-patterns to avoid

- **Deep nesting > 3 levels** — split into named transforms and `reference` them.
- **Lookup without `default`** — produces null → downstream concat produces `"null"` literal.
- **Multi-branch Velocity in `static.value`** — use `firstValid` + `conditional`/`dateCompare` instead; it's greppable and auditable.
- **Hardcoded `sourceId`** copied across transforms — prefer `sourceName` or (better) `reference` a named `accountAttribute` sub-transform.
- **Omitting `ignoreErrors: true`** on `firstValid` wrapping date logic — date parse failures will kill the whole identity refresh.
