# View Data Sets

| Property | Value |
|----------|-------|
| stepType | `"View Data Sets"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | All (i5, i7, i3) — no hardware restriction |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Envelope / `_biomekType` / `comArray` / structural rules: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

View Data Sets is a **design-time inspection step**: it lets the method author view the data sets,
labware properties, and global variables associated with a chosen piece of labware, and copy the
values to the clipboard (tab-delimited or CSV) for pasting into Notepad/Excel. Per the manual it
"aids beginning users in understanding data sets and how they are used [and] provides an easy means
of checking data set values at any point in the Biomek method" (manual 23-36 to 23-38). Its
enqueue and runtime behavior is a **no-op** — inserting the step performs no action and does no
runtime validation; all five parameters exist purely to persist the editor's inspection state. It
is safe to place anywhere between `Start` and `Finish`.

## Parameters Reference Table

### Step-Level Keys

All five keys are written and read **only** by the editor. They are therefore **inactive — not read at runtime**, but they *are* part of the persisted step configuration and every editor-produced step
carries them, so document/preserve them. Casing shown is the serialized form. Universal base
keys (`caption`, `disabled`, …) are not listed — see
[Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `labwareAt` | string | — | No | `""` | Deck position of the labware whose data sets are displayed. Editor-only; no runtime effect. |
| `depth` | string (integer) | stack index | No | `"0"` | **0-based stack depth from the top piece of labware at `labwareAt`.** `0` = the top plate; `1` = the plate directly under it; `2` = the plate under that; etc. Stored as a string. Editor-only. |
| `viewAll` | boolean | — | No | `false` (see note) | `true` = show **all** data sets on the labware; `false` = show only the single data set named by `singleDatasetName`. Editor-only. |
| `singleDatasetName` | string | — | No | `""` | Name of the single data set to display when `viewAll` is `false`; ignored when `viewAll` is `true`. Editor-only. |
| `copyCsvFormat` | boolean | — | No | `true` | Clipboard copy format toggle: `true` = CSV, `false` = tab-delimited. Editor-only. |

> **`viewAll` default note.** When the key is omitted, the fallback is `true`, but editor-produced steps carry an explicit `false` paired with a `singleDatasetName`. Prefer that shape to match the editor's output.

## Enumerated / Constrained Values

### `viewAll`

| Value | Meaning |
|-------|---------|
| `true` | Display all data sets attached to the selected labware ("View All Datasets"). |
| `false` | Display only the one data set named by `singleDatasetName` ("View One Dataset"). |

### `copyCsvFormat`

| Value | Meaning |
|-------|---------|
| `true` | Copy to clipboard as CSV (`.csv`). |
| `false` | Copy to clipboard as tab-delimited (for Excel). |

## Cross-Field Validation Rules

**None at enqueue.** The step performs no runtime validation and cannot fail a run on its own account.

The editor does surface two **design-time** messages while populating the inspection panel (shown in
the editor only, not raised at enqueue/run):

- A "specified dataset not found" message shown when `viewAll` is `false` and the named data set is absent. **Also fires when `depth` exceeds
  the stack size** — going past the bottom of the stack means no labware is resolved, so the same
  string is displayed.
- A "no datasets found" message shown when `viewAll` is `true` and no data sets are found.

## Structural Context

Leaf step. Do not include `subSteps` (omit, or an empty array).

## Canonical Example

Typical editor-produced shape (`viewAll: false` paired with a `singleDatasetName` such as `"Volume"`):

```json
{
  "stepType": "View Data Sets",
  "parameters": {
    "copyCsvFormat": true,
    "depth": "0",
    "labwareAt": "P5",
    "singleDatasetName": "Volume",
    "viewAll": false
  }
}
```

"View all data sets" variant (matches the omitted-key fallback):

```json
{
  "stepType": "View Data Sets",
  "parameters": {
    "copyCsvFormat": true,
    "depth": "0",
    "labwareAt": "P5",
    "singleDatasetName": "",
    "viewAll": true
  }
}
```
