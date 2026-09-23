# Finish

| Property | Value |
|----------|-------|
| stepType | `"Finish"` |
| Category | Boundary |
| Terminator | N/A |
| Compatible Hardware | All (no hardware restriction — inherits default) |

---

## Behavior Summary

The Finish step is the mandatory last step in every Biomek method. It marks the end of a method and performs post-run cleanup: discarding disposable tips from all pods (and washing fixed tips on Span-8 pods), parking pods and grippers at their safe home positions, clearing the software deck state of all labware, clearing external device (SILAS and pipettor) labware state, and clearing global variables. Each cleanup action is controlled by an independent boolean flag — any combination may be selected.

If the `report` flag is enabled, the Finish step enqueues a Reporting substep that writes labware data-set output to a text file, per-plate HTML/text files, or SQL Server. The Finish step also invokes registered method extensions.

Only the last Finish step in the method root actually executes its cleanup logic; if a Finish step is not the final substep of the method, the step returns immediately at enqueue. Users cannot delete, move, copy, or insert steps below the Finish step in the editor.

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `clearDeck` | `boolean` | — | No | `true` | Clear the software state of all labware from the instrument deck at method completion. Ensures the next method starts with a clean deck. **Always author this key.** |
| `clearDevices` | `boolean` | — | No | `true` | Clear the labware state from all SILAS and pipettor external devices. Ensures devices are empty for the next method. **Always author this key.** |
| `cleanupPods` | `boolean` | — | No | `true` | Unload disposable tips from all pods. For Span-8 pods, also performs a post-run wash of fixed tips. **Always author this key.** |
| `podsToMaxZ` | `boolean` | — | No | `true` | **Inactive — not read at runtime.** Defaults to `true` but is not read at enqueue — the pod parking behavior is actually controlled by `parkPods`. Preserve for round-trip; do not rely on it. |
| `parkPods` | `boolean` | — | No | `true` | Move all pods and grippers to their park (home) locations after method completion. Should be disabled if the method is used as a sub-method. Engine tolerates omission by falling back to `true`. |
| `clearGlobals` | `boolean` | — | No | `true` | Clear all global variables defined in the Start or Set Global steps. Prevents variable carry-over into subsequent methods. Engine tolerates omission. |
| `report` | `boolean` | — | No | `false` | Enable data-set reporting at method completion. When `true`, a Reporting substep is enqueued using the reporting keys below. Engine tolerates omission (default `false` means no reporting). |
| `path` | `string` | — | **Conditional (when `report=true`)** | `""` | File path or folder location for report output. Meaning depends on `type`: file path for `"Text File"`, folder for `"Per-Plate HTML Files"` / `"Per-Plate Text Files"`. **For `"SQL Server"` type, `path` holds the server hostname (the `Data Source` component of the built connection string).** Omission throws a "could not find Path" system error **only when `report=true`**. When `report=false`, may be omitted safely. |
| `type` | `string` | — | **Conditional (when `report=true`)** | `""` | Report style. See Enumerated Values section for allowed values. Gated on `report=true`. |
| `server` | `string` | — | No | `""` | **Display-only.** Saved to the step dictionary by the editor and shown in the print view, but has no effect on the actual SQL Server connection. The effective server hostname goes into `path`. Preserve verbatim for round-trip fidelity if present in an existing method; do not rely on it to configure the connection. |
| `catalog` | `string` | — | **Conditional (when `report=true`)** | `""` | SQL Server database (catalog) name. Gated on `report=true`. |
| `tableName` | `string` | — | **Conditional (when `report=true`)** | `""` | SQL Server table name for report data. The engine appends a timestamp suffix (`mm_dd_yyyy_hh_nn_ss`) at run time to avoid collisions across runs; the table actually created is `<tableName><mm_dd_yyyy_hh_nn_ss>`. Gated on `report=true`. |
| `userName` | `string` | — | **Conditional (when `report=true`)** | `""` | Database user name for SQL Server authentication. Gated on `report=true`. |
| `password` | `string` | — | **Conditional (when `report=true`)** | `""` | Database password for SQL Server authentication. Gated on `report=true`. |
| `authent` | `boolean` | — | **Conditional (when `report=true`)** | `false` | Authentication mode flag for SQL Server. Gated on `report=true`. |
| `connectionString` | `string` | — | **Conditional (when `report=true` and `type="SQL Server"`)** | `""` | OLEDB connection string for SQL Server. **For manually authored JSON, this field must be provided** — the engine uses this string to open the database connection; `userName` and `password` are used only when the Biomek editor builds the string interactively. An empty or missing string causes a database-connection validation failure. Example: `"Provider=SQLOLEDB.1;User ID=biomek_user;Password=****;Initial Catalog=BiomekData;Data Source=LABSERVER01"`. |
| `collapsed` | `boolean` | — | No | `true` | UI state flag controlling whether the reporting section is collapsed in the editor. Editor-written; not read at runtime. Not meaningful for method authoring. |
| `clearTool` | `boolean` | — | No | Not present | **Inactive — not read at runtime.** Do not author. Preserve verbatim if present in an existing method for round-trip fidelity.
| `extensions` | `object` | — | No | Not present | Container for registered method-extension data, keyed by extension class ID. Each extension receives its sub-dictionary during enqueue. Typically not authored manually. |
| `bitmap` | `string` | — | Yes | `"OStepUI.ocx,FINISH"` | Step-icon reference. Defaults to `"OStepUI.ocx,FINISH"`, but JSON import replaces the step's parameters wholesale and discards defaults — if omitted from authored JSON the Finish icon breaks on import. Author it exactly as shown. |

> _Other universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._
>
> _**Casing:** step dictionary keys are case-insensitive at runtime (see [00-Base-Keys §1](00-Base-Keys-and-Conventions.md#1-key-name-casing-spelling-global-rule)). Exports mix camelCase and all-lowercase spellings of the cleanup flags (e.g. `cleanupPods`/`cleanuppods`, `clearDeck`/`cleardeck`, `clearDevices`/`cleardevices`); both import identically. Author the camelCase form shown in this table._

---

## Enumerated / Constrained Values

### `type` (Report Style)

| Value | Description |
|-------|-------------|
| `"Text File"` | Single text file report. `path` specifies the full file path including file name. |
| `"Per-Plate HTML Files"` | One HTML file per plate. `path` specifies the output folder. |
| `"Per-Plate Text Files"` | One text file per plate. `path` specifies the output folder. |
| `"SQL Server"` | Report data is written to a SQL Server database. Requires `path` (server hostname), `catalog`, and `tableName`; requires `connectionString` when authoring JSON manually (the engine connects using only this string). `userName`, `password`, and `authent` are used by the editor to build `connectionString` interactively. |

These four are exactly the values offered by the Finish step's report-type combo. A fifth constant, `"Excel Spreadsheet"`, is defined in source but **is not exposed in the combo** — do not author it.

All other parameters use unconstrained boolean or string values.

---

## Cross-Field Validation Rules

1. All reporting keys (`path`, `type`, `server`, `catalog`, `tableName`, `userName`, `password`, `authent`, `connectionString`) are **only meaningful when `report` is `true`**. When `report` is `false`, they are ignored at runtime.
2. When `report` is `true`, `type` **must** be set to one of the four enumerated report styles. An empty or unrecognized `type` fails validation.
3. When `type` is `"SQL Server"`, `path` (server hostname), `catalog`, and `tableName` must be populated; `connectionString` must contain the full OLEDB connection string (the engine connects using this directly). An empty or unreachable connection string fails the connection check. A missing table name fails a distinct check. `userName` and `password` are used by the editor to build `connectionString` interactively but do not drive the connection themselves.
4. When `type` is `"Text File"`, `"Per-Plate HTML Files"`, or `"Per-Plate Text Files"`, `path` must point to a valid file/folder. For `"Text File"`: if the directory component of `path` does not exist, path validation fails; if the directory exists but `path` names an existing directory rather than a file, filename validation fails. For `"Per-Plate HTML Files"`: a non-existent directory fails path validation. For `"Per-Plate Text Files"`: a non-existent directory fails path validation. SQL-specific keys are ignored.
5. The five functional cleanup flags (`clearDeck`, `clearDevices`, `cleanupPods`, `parkPods`, `clearGlobals`) are **independent** — any combination may be selected.
6. If the method will be invoked as a sub-method via Run Method, set `parkPods` to `false` to avoid unnecessary pod movement between sub-method and parent method.
7. **`cleanupPods=true` requires a reachable trash ALP for every pod that has tips at Finish time.** If a pod is holding tips and no position with the `Can Discard Tips` characteristic is present (or reachable) for that pod, enqueue fails.
   Fix by placing/keeping a trash ALP the pod can reach, or by clearing the pod's tips before Finish.
8. **`parkPods=true` invokes the path planner and can fail when tall labware blocks the park path.** If a stack of labware makes the pod-park destination reachable only through a colliding configuration, enqueue fails with a path-planner collision error.
   Fix by moving the tall labware, choosing a different park configuration, or clearing tips so the gripper can retract before parking.

---

## Structural Context

The Finish step is a **boundary/anchor** step with the following structural constraints:

- **Must be the last element** of the method root `steps` array. The method is always structured as `[Start, ...body steps..., Finish]`.
- **Exactly one Finish step** per method. Duplicates are invalid.
- **Root-only**: The Finish step must never appear inside a container step's `subSteps`.
- **Cannot be deleted, moved, copied, or have steps inserted below it**.
- **Does not have `subSteps`** — the Finish step is not a container. Never include a `subSteps` array on it.
- **Auto-seeded**: When a new method is created, the method's substep-population routine automatically inserts a Start step at index 0 and a Finish step at index 1 with `bitmap: "OStepUI.ocx,FINISH"`.
- **Only executes if truly last**: At runtime, the Finish step checks whether this instance is the last substep of the method. If it is not the last substep, it returns immediately without performing any cleanup.

---

## Canonical Examples

### Minimal Finish (all cleanup defaults — the most common case)

All cleanup actions are enabled by default. Author `bitmap` explicitly (import replaces the constructor-seeded icon), and always author `clearDeck`, `clearDevices`, and `cleanupPods`.

```json
{
  "stepType": "Finish",
  "parameters": {
    "bitmap": "OStepUI.ocx,FINISH",
    "clearDeck": true,
    "clearDevices": true,
    "cleanupPods": true
  }
}
```

### Finish with selective cleanup (sub-method usage)

A Finish step configured for a method that will be called as a sub-method. Pod parking and global clearing are disabled to preserve state for the calling method:

```json
{
  "stepType": "Finish",
  "parameters": {
    "bitmap": "OStepUI.ocx,FINISH",
    "clearDeck": true,
    "clearDevices": true,
    "cleanupPods": true,
    "parkPods": false,
    "clearGlobals": false
  }
}
```

### Finish with text-file reporting

All cleanup enabled, plus a text-file report on labware data:

```json
{
  "stepType": "Finish",
  "parameters": {
    "bitmap": "OStepUI.ocx,FINISH",
    "clearDeck": true,
    "clearDevices": true,
    "cleanupPods": true,
    "parkPods": true,
    "clearGlobals": true,
    "report": true,
    "type": "Text File",
    "path": "C:\\Reports\\MethodReport.txt"
  }
}
```

### Finish with SQL Server reporting

Reporting to a SQL Server database with SQL authentication. `path` holds the server hostname (used by the editor to build `connectionString`); `connectionString` must be provided explicitly in manually authored JSON — the engine connects using only this field. The actual table created will be `<tableName><mm_dd_yyyy_hh_nn_ss>` (timestamp appended at run time):

```json
{
  "stepType": "Finish",
  "parameters": {
    "bitmap": "OStepUI.ocx,FINISH",
    "clearDeck": true,
    "clearDevices": true,
    "cleanupPods": true,
    "parkPods": true,
    "clearGlobals": true,
    "report": true,
    "type": "SQL Server",
    "path": "LABSERVER01",
    "server": "LABSERVER01",
    "catalog": "BiomekData",
    "tableName": "RunResults",
    "userName": "biomek_user",
    "password": "****",
    "authent": false,
    "connectionString": "Provider=SQLOLEDB.1;User ID=biomek_user;Password=****;Initial Catalog=BiomekData;Data Source=LABSERVER01"
  }
}
```

---

## Common Mistakes

- **Omitting `bitmap`** — Import replaces the step's parameters wholesale and discards the default icon value; without an explicit `"OStepUI.ocx,FINISH"` the Finish icon breaks on the next open.
- **Leaving `parkPods=true` in a sub-method** — When invoked via Run Method, parking between sub-method and parent wastes time and can perturb positioning.
- **Setting `report=true` without a `type`** — Fails validation. An empty `type` is not treated as "no report".
- **Confusing `clearDeck` with `clearDevices`** — `clearDeck` clears the software deck's labware state; `clearDevices` clears external device (SILAS, pipettor) labware state. They are independent, not redundant.
- **Trusting `podsToMaxZ`** — It's inactive: defaults to `true` but is not read at runtime. Preserve verbatim for round-trip fidelity, but do not expect it to control parking (that's `parkPods`).
- **Omitting `connectionString` from manually authored SQL Server JSON** — The engine connects using only `connectionString`; individual credential fields (`userName`, `password`) are used only when the Biomek editor builds the string interactively. Without an explicit `connectionString`, the step fails the database-connection check at validation.
- **Using `server` instead of `path` as the server hostname** — `server` is display-only and has no effect on the database connection. The server hostname must go in `path`; the editor reads `path` as the `Data Source` when building `connectionString`.
