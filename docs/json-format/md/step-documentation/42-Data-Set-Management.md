# Data Set Management

| Property | Value |
|----------|-------|
| stepType | `"Data Set Management"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7, i3 (hardware-independent data step) |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Envelope / `_biomekType` / structure: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Data Set Management step manipulates *existing* data sets attached to a single
piece of labware — it neither creates nor computes data (that is the job of Create Data
Set and Data Set Processing). At enqueue it resolves the labware at `position`/`depth`,
retrieves that labware's `DataSets` collection, and performs one of four operations
selected by `operation`: **Copy** a data set to a new name, **Rename** a data set,
**Remove** a data set, or **Change** the two flags (`reportable`, `copyOnTransfer`) on a
data set. `Copy` and `Rename` require both a `source` and a `destination` name;
`Remove` and `Change` use only `source`. Use it to duplicate/rename the `Sample ID`
data set before a second Instrument Setup overwrites it, to clear data sets that are no
longer needed, or to toggle whether a data set appears in reports / is carried during
pipetting. The step runs entirely in the enqueue (method-build) phase; it performs no runtime
actions. (Manual: *Configuring the Data Set Management Step*, ch. 23.)

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `position` | expression-capable string | — | Yes | *(none — checked)* | Deck position of the labware holding the data sets (e.g., `"P1"`). Evaluated as an expression. |
| `depth` | expression-capable string / integer | — | No | `0` | Stack depth at `position`. `0` = top piece of labware, `1` = second from top, etc. Evaluated as an expression. |
| `operation` | string (enum) | — | Yes | *(none — checked)* | Which management action to perform: `"Copy"`, `"Rename"`, `"Remove"`, or `"Change"`. Matched **case-insensitively** (see Enumerated Values). |
| `source` | expression-capable string | — | Yes | *(none — checked)* | Name of the data set to act on (copy/rename/remove/change). Evaluated as an expression. |
| `destination` | expression-capable string | — | Conditional | `""` | New data-set name. **Required** for `Copy` and `Rename`; ignored for `Remove` and `Change`. Evaluated as an expression. |
| `reportable` | boolean | — | Conditional | `true` | **`Change` only:** sets the data set's `reportable` flag (include in reports). Ignored for other operations. |
| `copyOnTransfer` | boolean | — | Conditional | `true` | **`Change` only:** sets the data set's `copyOnTransfer` flag (data tracked during pipetting). **Not applied to the built-in `Volume` data set.** Ignored for other operations. |

The editor writes all seven keys on every save.

Universal base keys (`caption`, `dynamic?`, `disabled`, …) — see
[Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md). This step has no
item array and no editor-only state keys of its own.

## Enumerated / Constrained Values

### operation

The enqueue matches operation strings case-insensitively; the editor writes **title-case**, which is the form real exports carry. Prefer title-case.

| Value | Meaning | Keys used |
|-------|---------|-----------|
| `"Copy"` | Duplicate `source` → new data set `destination`. | `source`, `destination` |
| `"Rename"` | Rename `source` → `destination`. | `source`, `destination` |
| `"Remove"` | Delete the `source` data set. | `source` |
| `"Change"` | Set the `reportable` / `copyOnTransfer` flags on `source`. | `source`, `reportable`, `copyOnTransfer` |

> Author these in title case (`"Copy"`). Matching is case-insensitive, so `"COPY"`
> imports too, but title case is the form the editor writes — use it.

## Cross-Field Validation Rules

1. `position`, `operation`, and `source` must each be bound.
2. For `operation` = `Copy` or `Rename`, `destination` must be bound.
3. `position` must evaluate to a real deck position.
4. Labware must exist at the requested depth.
5. `operation` must be one of the four recognized values.
6. `Copy`/`Rename` will fail downstream if `destination` collides with an existing data set — each data set must have a unique name.

## Structural Context

Leaf step. Do not include `subSteps` (or include an empty array).

## Canonical Example

Rename the `Sample ID` data set on labware at P1:

```json
{
  "stepType": "Data Set Management",
  "parameters": {
    "position": "P1",
    "depth": "0",
    "operation": "Rename",
    "source": "Sample ID",
    "destination": "Sample ID Run 1",
    "reportable": true,
    "copyOnTransfer": true
  }
}
```

Change the flags on a computed data set (Change operation):

```json
{
  "stepType": "Data Set Management",
  "parameters": {
    "position": "P1",
    "depth": "0",
    "operation": "Change",
    "source": "Concentration",
    "destination": "",
    "reportable": true,
    "copyOnTransfer": false
  }
}
```
