# Method JSON Structure

**Scope**: Structure and value-encoding rules only — up to but NOT including how to populate step-specific parameter values. For per-step parameter reference, see [step-documentation/](step-documentation/).

**Prerequisite**: [Introduction to Biomek and Liquid Handling Method Writing](Introduction-to-Biomek-Method-JSON.md).

> **Step documentation scope**: Step-specific parameter documentation is available for **Documented** steps only. When generating methods, **use only steps marked "Documented" in the step catalog below**. Other step types are not covered by this documentation set and may have unpublished structural or parameter requirements.

---

## JSON Envelope

Every method file is a JSON object with three required top-level properties:

```jsonc
{
  "format": "Biomek Method",
  "formatVersion": "1.0",
  "author": "",
  "description": "",
  "steps": [ ... ]
}
```

| Property | Required | Type | Description |
|----------|----------|------|-------------|
| `format` | Yes | string | Must equal `"Biomek Method"` (case-insensitive comparison). |
| `formatVersion` | Yes | string | The Biomek **Method** JSON format version. Format should be major.minor (e.g. `"1.0"`). |
| `author` | No | string | Free-text method author. Round-trips to/from the method's `Author` dictionary key. Missing or `null` becomes an empty string on import; always written on export. A non-string JSON value (e.g. a number or object) is a deserialization error. |
| `description` | No | string | Free-text method description. Round-trips to/from the method's `Description` dictionary key. Missing or `null` becomes an empty string on import; always written on export. A non-string JSON value (e.g. a number or object) is a deserialization error. |
| `steps` | Yes | array | Ordered list of root-level step nodes. |

`format`, `formatVersion`, and `steps` are required — omitting any fails import with a hard error. `author` and `description` are optional. `format` and `formatVersion` are also verified in a first-pass check before schema-level parsing, so their errors surface first; a version mismatch is reported as too-old or too-new for the running software (see the `formatVersion` row above for the exact string rules).

The `steps` array holds the direct children of the method root: the top level of the method tree.

---

## Step Node Structure

Each element in `steps` (and in any nested `subSteps` array) is a **step node**:

```jsonc
{
  "stepType": "Transfer",
  "disabled": false,
  "parameters": { },
  "subSteps": [ ... ]
}
```

| Property | Required | Type | Description |
|----------|----------|------|-------------|
| `stepType` | Yes | string | Step type name — see step catalog. Matched case-insensitively against the step registry or the step's ProgID / CLSID string. |
| `disabled` | No | boolean | Whether the step is commented out (disabled) in the editor. Defaults to `false` and is omitted from export when `false`. A `disabled` value other than `true` or `false` fails deserialization. When `true`, the step is skipped at run time but preserved in the tree. Author disabled state only via this step-node property; do not author an equivalent flag inside `parameters`. |
| `parameters` | Yes | object | Case-insensitive key-value map of step configuration. May be `{}` but the property must be present. |
| `subSteps` | No | array | Child step nodes. Omit (or set to `null`) for leaf steps. An empty array is also accepted. `subSteps` is written only when the exporter has one to emit. |

### Property Naming Convention

Envelope and step-node property names (`format`, `formatVersion`, `steps`, `stepType`, `parameters`, `subSteps`) use **camelCase**. Parameter keys follow the casing rule documented in [Base Keys & Serialization Conventions](step-documentation/00-Base-Keys-and-Conventions.md#1-key-name-casing-spelling-global-rule) §1, which includes the full worked-example table. On read, envelope and step-node property names are matched case-insensitively; parameter keys are matched case-insensitively too — **except within SILAS message payloads, which are case-sensitive.**

---

## JSON Parsing Behavior

Biomek's JSON reader/writer uses the options below — they affect what counts as valid JSON on import:

| Behavior | Setting | Effect |
|---|---|---|
| Duplicate JSON keys | Rejected | Duplicate keys in the same JSON object cause the parser to fail. |
| Trailing commas | Allowed | A trailing comma at the end of an array or object is accepted. |
| Property-name matching | Case-insensitive | Envelope and step-node field names are matched case-insensitively on read. |
| Naming on write | camelCase | Envelope fields are written in camelCase. Parameter keys are converted by the same rule: the leading run of uppercase letters is lowercased except the last one (when a lowercase follows), and the rest is preserved verbatim (see [Base Keys & Serialization Conventions](step-documentation/00-Base-Keys-and-Conventions.md#1-key-name-casing-spelling-global-rule) §1). |
| Comments | Allowed and ignored | `//` line comments and `/* */` block comments are accepted and stripped on read. |
| Null values | Rejected for non-nullable properties | A JSON `null` for a property that is not declared nullable is rejected on read. Property *presence* is enforced separately, on `format`/`formatVersion`/`steps`/`stepType`/`parameters`. |
| Output formatting | Pretty-printed | Written JSON is indented for readability. |

`double` values are written by a custom converter that forces a decimal point on integral values (`3` → `3.0`) so a round-trip preserves double-vs-integer typing. Numeric reading is locale-invariant.

---

## Parameter Value Encoding

The `parameters` object is a JSON representation of the step's parameter map — a case-insensitive key/value collection. Most values are ordinary JSON primitives (`string`, `number`, `boolean`, `null`, or nested `object`/`array`), but some Biomek types require a **`_biomekType` discriminator** — a reserved property on the JSON object that identifies its Biomek type at deserialization time.

### `_biomekType` Discriminator

Any JSON object inside `parameters` may carry a `_biomekType` string property (matched case-insensitively) that selects one of the following providers:

| `_biomekType` value | Represents | Notes |
|---|---|---|
| *(absent or empty string)* | Plain nested dictionary | The default. A JSON object without `_biomekType` is treated as a nested dictionary. Keys are case-insensitive. |
| `"dateTime"` | Date/time value | Serialized as `{ "_biomekType": "dateTime", "value": "2024-01-31T13:45:30.0000000Z" }` — ISO 8601 round-trip (`"O"`) format, locale-invariant. |
| `"empty"` | Absent / unset value (distinct from `null`) | Distinct from `null` in COM semantics. |
| `"missing"` | Parameter not supplied | Rarely authored by hand. |
| `"comArray"` | Typed array | See "comArray" section below. |
| `"nonFiniteNumber"` | A non-finite `double`/`float` (`NaN`, `Infinity`, `-Infinity`) | JSON numbers can't express these, so they serialize as `{ "_biomekType": "nonFiniteNumber", "value": "NaN" }`. `value` must be the string `"NaN"`, `"Infinity"`, or `"-Infinity"` (case-insensitive). Finite numbers use plain JSON numbers. |
| `"labware"` | Serialized labware reference | Used inside Instrument Setup and similar steps. |
| `"technique"` | Custom pipetting technique object | An inline technique definition, distinct from a technique referenced by name. |
| `"SILASMessage"` | SILAS device/consumer message payload | Case-**sensitive** keys inside — SILAS payloads preserve casing exactly. |

Rule: **You must not use `_biomekType` as your own key** anywhere inside `parameters`. Attempting to serialize a step whose dictionary contains a `_biomekType` key raises a `JsonException` up front — the exporter refuses to hide the discriminator behind user data.

On import, a JSON object whose `_biomekType` value does not match any provider fails deserialization.

### `comArray` required shape

Any typed Biomek COM array serializes as:

```jsonc
{
  "_biomekType": "comArray",
  "arraySubtype": "<subtype>",
  "values": [ ... ]
}
```

Legal `arraySubtype` values:

| `arraySubtype` | Element type | Example use |
|---|---|---|
| `"boolean"` | `bool` | Well-selection masks, probe-active arrays |
| `"byte"` | `byte` | Binary payloads |
| `"dateTime"` | `DateTime` | Timestamp arrays |
| `"double"` | `double` | Per-well volumes |
| `"integer"` | `int` (32-bit) | Section indices |
| `"short"` | `short` (16-bit) | 16-bit integer indices |
| `"single"` | `float` | Single-precision floats |
| `"string"` | `string` | Text arrays |
| `"variant"` | `object` | Mixed-type arrays |

If you emit a plain JSON array (`[true, false, ...]`) where a `comArray` is required, deserialization will treat it as a plain list — a different Biomek type — and downstream step logic may reject it or interpret it incorrectly. When in doubt, follow the shape shown in the step's documented examples.

#### Complex (multidimensional / non-zero-based) comArrays

Most `comArray` values are one-dimensional and zero-based, needing only `arraySubtype` and `values`. Two optional properties preserve unusual COM array shapes:

| Property | When present | Meaning |
|---|---|---|
| `dimensionCount` | Written only when the array has more than one dimension | Number of dimensions (2–32). When present, `values` is a nested array matching that rank. |
| `lowerBound` | Written only when at least one dimension does not start at index 0 | Either a single integer (all dimensions share the lower bound) or an array of integers (one per dimension). |

You will rarely author these by hand — no built-in step is known to use them — but they appear when round-tripping methods authored by custom step plug-ins. Preserve them verbatim if you see them. A multidimensional array may have zero length only in its final dimension.

### Plain nested dictionary

A JSON object with no `_biomekType` is a plain nested dictionary. Keys are case-insensitive; on write, each key is converted by the camelCase rules described in [Base Keys & Serialization Conventions](step-documentation/00-Base-Keys-and-Conventions.md#1-key-name-casing-spelling-global-rule) §1. Nesting is supported to a depth of 500, far deeper than any real method.

### Locale, `null`, and non-finite numbers

- Numeric formatting is locale-invariant (`.` decimal separator always). Finite `double` values are always written with a decimal point (`10` → `10.0`) so a round-trip preserves double-vs-integer typing.
- JSON `null` represents a null value; `_biomekType: "empty"` represents an absent/unset value; both are distinct from each other and from a missing key.
- Non-finite doubles (`NaN`, `Infinity`, `-Infinity`) serialize as a `_biomekType: "nonFiniteNumber"` object (see the discriminator table above) — plain JSON does not support them as numbers.

---

## Step Types and Structural Kinds

The core step registry defines the built-in step types. `stepType` matching is case-insensitive; prefer the canonical spelling. Steps contributed by add-on modules and device integrations are documented separately in [`add-on-steps/`](add-on-steps/index.md).

Steps fall into a few **structural kinds**:

- **Leaf** — no child steps.
- **Free-form container** — a user-arranged list of children, closed by a **terminator** (the fixed trailing anchor child, usually `"End"`).
- **Exact-children container** — a fixed set of specific children in a defined order.

> Structure classifications are derived from the step's runtime editor-options flags plus its factory-default children. The catalog reflects the current implementation; the running software is authoritative.

For the full list of step types and per-step docs, see the [step documentation index](step-documentation/index.md).

> **Unknown `stepType` names** — Unrecognized names fall through to a CLSID-string parse and then a ProgID lookup; if nothing resolves, import fails. Structural validation gives such unknown types "permissive" rules (no additional constraints), so the meaningful error comes from the deserializer rather than the validator.

---

## Beyond the Built-in Steps (Add-on / Plugin Steps)

The [step documentation index](step-documentation/index.md) covers the core step registry, and [`add-on-steps/`](add-on-steps/index.md) covers the documented add-on steps. Add-on modules, device integrations, and site plug-ins register their own step types beyond those two sets — handle those as follows:

- **An add-on step's `stepType` may be a friendly name or a COM ProgID, depending on the add-on.** A step registered by a plug-in that has not published a JSON type name carries its registration identifier (a ProgID) as `stepType`. Import resolves the name against the step registry first, then a CLSID-string parse, then a ProgID lookup (see the *Unknown `stepType` names* note above), so a ProgID string is a legal `stepType`.
- **Do not guess parameters for a step type, device, or integration this documentation does not cover.** A `stepType` outside the step and add-on indexes — or a device or integration not described here — may carry unpublished structural or parameter requirements. Authoring one from a guess risks a method that fails import, or one that imports and then runs incorrectly.
- **Escalate for the authoring spec.** When a method must use an add-on step outside this set, obtain its parameter reference from Beckman Coulter — or, for a third-party module, the add-on's vendor — rather than inferring it.

---

## Structural Rules

Every step falls into one of three structural shapes, which determine what its `subSteps` may contain:

- **Leaf**: takes no child steps. Omit `subSteps` (or provide an empty array `[]`).
- **Fixed-children container**: requires an exact set of children — a specific count and type, in order — and `subSteps` must match that set exactly. Example: `"If"` requires exactly 2 `"Named Container"` children.
- **Free-form container**: accepts any legal steps in the middle, but its fixed **anchor** steps must stay put — any leading anchors come first and any trailing anchors come last, in their defined order.

### Key Rules Summary

1. **Method root** is a free-form container with a leading `"Start"` (top anchor) and a trailing `"Finish"` (bottom anchor): root children must have `"Start"` first and `"Finish"` last.
2. **A free-form container's terminator is whatever step type that container defines** as its trailing bottom-anchor child — it is not always `"End"`:
   - Most built-in containers (`"Group"`, `"Loop"`, `"Let"`, `"Scripted Let"`, `"Hold Labware"`, `"Named Container"`, `"Just In Time"`, `"Worklist"`, `"Define Procedure"`) use `"End"`. The editor caption varies (`"End Group"`, `"End Loop"`, `"End Just In Time"`, …) but that is display text only — the step registry registers a single `"End"` for the whole group, and serializing `"stepType": "End Group"` fails import.
   - The `"Multichannel Select Tips"` family ends with `"Multichannel Select Tips End"`.
   - **Add-on / plugin-registered steps may register their own terminator types**. Match the terminator to what the specific container populates rather than assuming a fixed set.
3. **`"If"`** is an exact-children container with exactly 2 `"Named Container"` children (Then, Else). Each Named Container is itself a free-form container ending in `"End"`.
4. **Leaf steps** must not have `subSteps` with elements. Omitting or providing `[]` is equivalent.
5. **Anchors and terminators** (`"Start"`, `"Finish"`, and each container's own terminator such as `"End"` or `"Multichannel Select Tips End"`) must appear only in their designated positions — never as free user-inserted steps mid-body.

---

## Minimal Valid Method

The simplest possible valid method:

```json
{
  "format": "Biomek Method",
  "formatVersion": "1.0",
  "steps": [
    { "stepType": "Start", "parameters": { "bitmap": "OStepUI.ocx,START", "let": {}, "weak": {}, "prompt": {} } },
    { "stepType": "Finish", "parameters": { "bitmap": "OStepUI.ocx,FINISH" } }
  ]
}
```

This is an empty method — it has only the required anchors and does nothing. Author `Start` with `let`/`weak`/`prompt` dictionaries (empty `{}` is fine) and `bitmap: "OStepUI.ocx,START"`; author `Finish` with `bitmap: "OStepUI.ocx,FINISH"`. All four are required because JSON import replaces the step's parameters wholesale, discarding the built-in defaults — an omitted key hard-fails at Run (for `let`/`weak`/`prompt`) or leaves the anchor icon broken (for `bitmap`). See [Base Keys & Serialization Conventions](step-documentation/00-Base-Keys-and-Conventions.md#4-keys-that-look-shared-but-are-genuinely-per-step) §4 (environment-extension keys) and §2 (`bitmap`); on every other step `bitmap` is auto.

---

## Building Up a Method

### Adding a Simple Step

Insert operational steps between `Start` and `Finish`:

```jsonc
{
  "format": "Biomek Method",
  "formatVersion": "1.0",
  "steps": [
    { "stepType": "Start", "parameters": { "bitmap": "OStepUI.ocx,START", "let": {}, "weak": {}, "prompt": {} } },
    { "stepType": "Transfer", "parameters": { ... } },
    { "stepType": "Finish", "parameters": { "bitmap": "OStepUI.ocx,FINISH" } }
  ]
}
```

### Adding a Free-form Container (Group)

The `Group` container uses `subSteps` and ends with an `"End"` step:

```jsonc
{
  "format": "Biomek Method",
  "formatVersion": "1.0",
  "steps": [
    { "stepType": "Start", "parameters": { "bitmap": "OStepUI.ocx,START", "let": {}, "weak": {}, "prompt": {} } },
    {
      "stepType": "Group",
      "parameters": {},
      "subSteps": [
        { "stepType": "Span-8 Aspirate", "parameters": { ... } },
        { "stepType": "Span-8 Dispense", "parameters": { ... } },
        { "stepType": "End", "parameters": {} }
      ]
    },
    { "stepType": "Finish", "parameters": { "bitmap": "OStepUI.ocx,FINISH" } }
  ]
}
```

### Adding a Loop

```jsonc
{
  "format": "Biomek Method",
  "formatVersion": "1.0",
  "steps": [
    { "stepType": "Start", "parameters": { "bitmap": "OStepUI.ocx,START", "let": {}, "weak": {}, "prompt": {} } },
    {
      "stepType": "Loop",
      "parameters": {},
      "subSteps": [
        { "stepType": "Span-8 Aspirate", "parameters": { ... } },
        { "stepType": "Span-8 Dispense", "parameters": { ... } },
        { "stepType": "End", "parameters": {} }
      ]
    },
    { "stepType": "Finish", "parameters": { "bitmap": "OStepUI.ocx,FINISH" } }
  ]
}
```

### Adding a Conditional (If)

Exact-children container — exactly 2 `"Named Container"` children:

```jsonc
{
  "format": "Biomek Method",
  "formatVersion": "1.0",
  "steps": [
    { "stepType": "Start", "parameters": { "bitmap": "OStepUI.ocx,START", "let": {}, "weak": {}, "prompt": {} } },
    {
      "stepType": "If",
      "parameters": {},
      "subSteps": [
        {
          "stepType": "Named Container",
          "parameters": {},
          "subSteps": [
            { "stepType": "Transfer", "parameters": { ... } },
            { "stepType": "End", "parameters": {} }
          ]
        },
        {
          "stepType": "Named Container",
          "parameters": {},
          "subSteps": [
            { "stepType": "Pause", "parameters": { ... } },
            { "stepType": "End", "parameters": {} }
          ]
        }
      ]
    },
    { "stepType": "Finish", "parameters": { "bitmap": "OStepUI.ocx,FINISH" } }
  ]
}
```

### Nesting Containers

Containers can be nested arbitrarily deep:

```jsonc
{
  "stepType": "Loop",
  "parameters": {},
  "subSteps": [
    {
      "stepType": "If",
      "parameters": {},
      "subSteps": [
        {
          "stepType": "Named Container",
          "parameters": {},
          "subSteps": [
            {
              "stepType": "Group",
              "parameters": {},
              "subSteps": [
                { "stepType": "Transfer", "parameters": { ... } },
                { "stepType": "End", "parameters": {} }
              ]
            },
            { "stepType": "End", "parameters": {} }
          ]
        },
        {
          "stepType": "Named Container",
          "parameters": {},
          "subSteps": [
            { "stepType": "End", "parameters": {} }
          ]
        }
      ]
    },
    { "stepType": "End", "parameters": {} }
  ]
}
```

---

## Structural Validation Errors

When structural validation fails, the importer reports every violation and raises a single error covering all of them. Each violation is annotated with a breadcrumb `location` identifying the offending step in the tree, and a message describing the specific rule broken. Categories of validation checked:

- Free-form container leading/trailing sequences (anchor position and count).
- Leaf and exact-children container child counts and types.
- Per-step rules, including the anchor-cannot-be-disabled rule (a step carrying `disabled: true` that is a structural anchor — `Start`, `Finish`, or a container's `End` terminator — is rejected).

Envelope errors (missing `format` / `formatVersion`, malformed version string, wrong `format` value, or a `formatVersion` that is not the current one — rejected as too old or too new) are raised before structural validation, so a bad envelope surfaces first.

---

## Composition Checklist

When constructing a method, verify:

1. Envelope has `format = "Biomek Method"`, the current `formatVersion`, and a `steps` array.
2. Root `steps` starts with `"Start"` and ends with `"Finish"`.
3. No `"Start"` or `"Finish"` inside any container — they belong only at the method root. Author the required anchor keys on each (see the *Minimal Valid Method* section above).
4. Every `"If"` has exactly 2 `"Named Container"` children.
5. Every free-form container's `subSteps` ends with the terminator that container defines (`"End"` for most, `"Multichannel Select Tips End"` for the Select-Tips family, or an add-on step's own terminator type).
6. No leaf step has `subSteps` with elements.
7. Every `parameters` object with COM-array or COM-object values uses the correct `_biomekType` discriminator and shape.
8. No parameter dictionary contains a `_biomekType` key of your own.
9. Only Documented steps from the catalog are used.

If all checks pass, the method structure is valid.

---

## Next: populating step parameters

This document covers method *structure* — the tree of steps. Once that structure is valid, each step's `parameters` still need the correct keys and values for its step type. For those, see the per-step reference in [step-documentation/](step-documentation/) and the [Pipetting Techniques, Templates, and Auto-Selection](Pipetting-Techniques-and-Templates.md) reference.
