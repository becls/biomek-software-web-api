# Data Set Processing

| Property | Value |
|----------|-------|
| stepType | `"Data Set Processing"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7, i3 (hardware-independent data step) |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Envelope / `_biomekType` / structure: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Data Set Processing step applies a **transformation expression** to an existing
(source) data set to build a **new** (destination) data set on the same labware. At
enqueue it resolves the labware at `position`/`depth`, adds a data set named
`destination`, and evaluates `expression` once per element (well or tube). When `source`
is non-empty, the step iterates the source data set: for each element whose value is
**not empty**, it binds the source data set's name as an expression variable (equal to
that element's value), evaluates `expression`, and writes the result to the matching
element of the destination — empty source elements are skipped. When `source` is left
empty, the step computes without a source, sizing the destination from the labware
geometry (reservoir section count, else `WellsX × WellsY`) and evaluating `expression`
for every element. The result is whatever the expression evaluates to — commonly numeric,
Boolean (`True`/`False` in reports), string, or date/time. Finally it sets the destination's `reportable` and `copyOnTransfer` flags.
Use it to derive concentrations, dilution factors, pass/fail masks, etc. from measured
data. (Manual: *Configuring the Data Set Processing Step*, ch. 23.)

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `position` | expression-capable string | — | Yes | *(none — checked)* | Deck position of the labware where the new data set is created. Evaluated as an expression. |
| `depth` | expression-capable string / integer | — | No | `0` | Stack depth at `position` (`0` = top). Evaluated as an expression. |
| `expression` | expression string | — | Yes | *(none — checked; also must be non-empty)* | Transformation applied per element. **Must be prefixed with `=`** (VBScript) or `==` (JScript). May reference the source data set by its **name** as a variable (e.g., `"=Concentration * 1.05"`). |
| `source` | expression-capable string | — | **Yes** | *(the key must exist)* | Name of the existing data set to transform. The key itself **must be present** — omitting it fails at enqueue. Its **value may be empty** to compute a fresh data set from labware geometry (no source binding). When non-empty, only non-empty source elements produce destination values. |
| `destination` | expression-capable string | — | Yes | *(none — checked; must be non-empty)* | Name of the new data set to create. Evaluated as an expression. |
| `reportable` | boolean | — | No | `true` | Sets the new data set's `reportable` flag (include in reports). |
| `copyOnTransfer` | boolean | — | No | `true` | Sets the new data set's `copyOnTransfer` flag (data tracked during pipetting). |

The editor writes all seven keys on every save. All are scalar; this step has no item array and no
editor-only state keys of its own.

Universal base keys (`caption`, `dynamic?`, `disabled`, …) — see
[Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).

## Enumerated / Constrained Values

No enumerated (magic-string) keys. `expression` is free-form VBScript/JScript.

**Trap:** without the leading `=`/`==` the expression is treated as a literal and will
not compute. Common VBScript operators:
`+ - * / \ (integer div) MOD > < = <> ^`. See manual Table 23.2–23.5.

## Cross-Field Validation Rules

1. `position`, `expression`, and `destination` must be bound. **Note:** `source` is not part of that upfront
   bound-key check, but the key must still be present because it is read via a strict dictionary lookup that
   fails when missing. Its **value** may be the empty string to invoke the no-source branch (see `source`
   parameter description).
2. `position` must resolve.
3. Labware must exist at depth.
4. `destination` must evaluate to a non-empty string.
5. `expression` must be non-empty.
6. Duplicate destination names are rejected during validation — each data set on a
   labware must have a unique name.

## Structural Context

Leaf step. Do not include `subSteps`.

## Canonical Example

Create `Concentration` from an existing `Absorbance` data set:

```json
{
  "stepType": "Data Set Processing",
  "parameters": {
    "position": "P2",
    "depth": "0",
    "source": "Absorbance",
    "destination": "Concentration",
    "expression": "=Absorbance * 1.05",
    "reportable": true,
    "copyOnTransfer": true
  }
}
```

Compute a constant data set with no source (fills every well):

```json
{
  "stepType": "Data Set Processing",
  "parameters": {
    "position": "P2",
    "depth": "0",
    "source": "",
    "destination": "InitialVolume",
    "expression": "=100",
    "reportable": true,
    "copyOnTransfer": false
  }
}
```
