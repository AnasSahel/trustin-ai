# ISC Transform Types — Complete Reference

All transform types accepted by the SailPoint ISC v3/v2025 transforms API. Grouped by category. Unknown types are silently rejected (runtime value = null), so always pick from this list.

Every transform has the shape:
```json
{ "name": "...", "type": "<from below>", "attributes": { ... } }
```

Any attribute that accepts input can take either a literal value or a nested transform object.

## String operations

| type | purpose | required attrs | notes |
|---|---|---|---|
| `lower` | lowercase | `input` | |
| `upper` | UPPERCASE | `input` | |
| `trim` | strip leading/trailing whitespace | `input` | |
| `concat` | join strings end-to-end | `values` (array) | elements can be literals or transforms |
| `join` | join with separator | `values`, `separator` | |
| `split` | pick one element from split | `input`, `delimiter`, `index` | `index` is 0-based; use `-1` for last |
| `substring` | extract range | `input`, `begin` | `end` optional (omit = to end) |
| `replace` | regex substitution | `input`, `regex`, `replacement` | single regex |
| `replaceAll` | multi-pattern substitution | `input`, `table` (object) | `table` maps regex → replacement |
| `indexOf` | position of first match | `input`, `substring` | returns -1 if not found |
| `lastIndexOf` | position of last match | `input`, `substring` | |
| `getEndOfString` | last N chars | `input`, `numChars` | |
| `leftPad` | pad left to length | `input`, `length`, `padding` | |
| `rightPad` | pad right to length | `input`, `length`, `padding` | |
| `decomposeDiacriticalMarks` | strip accents (é→e, ñ→n) | `input` | doesn't touch case |
| `normalizeNames` | title-case + strip diacritics | `input` | SailPoint built-in name cleaner |
| `base64Encode` | encode | `input` | |
| `base64Decode` | decode | `input` | |

## Date operations

| type | purpose | required attrs | notes |
|---|---|---|---|
| `dateFormat` | reformat date string | `input`, `inputFormat`, `outputFormat` | formats use Java SimpleDateFormat or `ISO8601` |
| `dateMath` | add/subtract/round | `input`, `expression` | expr: `+1d`, `-30d/d`, `now`, `+1M/M`; `roundUp` optional |
| `dateCompare` | compare two dates | `firstDate`, `secondDate`, `operator`, `positiveCondition` | operators: `LT`, `LTE`, `GT`, `GTE`, `EQ`; returns `positiveCondition` if true, else passes through next (used inside `firstValid`) |

## Conditional / control flow

| type | purpose | required attrs | notes |
|---|---|---|---|
| `conditional` | if/then/else | `expression`, `positiveCondition`, `negativeCondition` | `expression` is `"$var eq 'X'"` style; needs variable bindings as sibling attrs |
| `firstValid` | first non-null | `values` (array) | `ignoreErrors: true` to swallow exceptions from inner transforms |

## Identity / account attributes

| type | purpose | required attrs | notes |
|---|---|---|---|
| `identityAttribute` | read attribute from current identity | `name` | also used as `attributes.name` form |
| `accountAttribute` | read from a specific source account | `sourceName`, `attributeName` | `sourceId` optional but preferred for stability |
| `getReferenceIdentityAttribute` | read attribute from another identity | `uid`, `attributeName` | `uid` is the referenced identity's uid |
| `displayName` | built-in SailPoint display-name formatter | `input` | usually `input: "input"` inside `concat` |

## Lookup / reference

| type | purpose | required attrs | notes |
|---|---|---|---|
| `lookup` | key→value table | `table` (object) | always include `"default"` key |
| `reference` | call another named transform | `id` (transform name) | `input` optional (passes through) |
| `rule` | invoke a cloud rule | `name`, `operation` | `operation: "calculate"` usually |

## Generators

| type | purpose | required attrs |
|---|---|---|
| `randomAlphaNumeric` | random mixed-case alnum | `length` |
| `randomNumeric` | random digits | `length` |
| `generateRandomString` | configurable random | `length`, `includeNumbers`, `includeSpecialChars` |
| `uuid` | UUID v4 | none |
| `usernameGenerator` | unique username w/ collision retry | `sourceCheck`, `transforms` |

## Formatters

| type | purpose | required attrs |
|---|---|---|
| `static` | fixed value (supports Velocity in `value`) | `value` |
| `e164phone` | normalize phone to E.164 | `input`, `defaultRegion` (ISO2) |
| `iso3166` | country code/name normalize | `input`, `format` (`alpha2`, `alpha3`, `numeric`) |
| `rfc5646` | language tag normalize | `input` |

## Velocity inside `static`

The `value` of a `static` transform supports Apache Velocity templating. Variable bindings are any *other* keys you add to `attributes`. Example:

```json
{
  "type": "static",
  "attributes": {
    "firstName": { "type": "identityAttribute", "attributes": { "name": "firstname" } },
    "lastName":  { "type": "identityAttribute", "attributes": { "name": "lastname" } },
    "value": "$firstName.substring(0,1).toLowerCase()$lastName.toLowerCase()"
  }
}
```

Keep Velocity to simple expressions and single-line `#if`. If you need multi-branch logic, use nested `firstValid`/`conditional`/`dateCompare` instead — it's easier to audit.
