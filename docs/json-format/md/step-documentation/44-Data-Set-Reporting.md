# Data Set Reporting

| Property | Value |
|----------|-------|
| stepType | `"Data Set Reporting"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7, i3 (hardware-independent data step) |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Envelope / `_biomekType` / structure: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Data Set Reporting step generates a report of every reportable data set on every
piece of labware known to the world — it sweeps labware on the deck, in stacker/carousel
stacks, on SILAS devices, and in Cytomat storage, skips tip boxes and any labware class
tagged `Do Not Report`, and hands them to the world's `ReportGenerator`. It then writes
the report in the format chosen by `type`: a single **Text File**, **Per-Plate HTML
Files** or **Per-Plate Text Files** (one file per plate in a directory), an **Excel
Spreadsheet** (`.xls`; see caveat), or a **SQL Server** table. For SQL, the step opens
the connection built from the UI fields (or a literal `connectionString`), appends a
date-time stamp to `tableName`, and writes the table. The `Volume` data set and any
data set with reporting enabled (see Data Set Management / Processing `reportable` flag)
appear in the output. Insert it after transfer/pipetting operations to snapshot well
volumes and computed data sets. Triggered at enqueue. (Manual: *Configuring the
Data Set Reporting Step*, ch. 23.)

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `type` | string (enum) | — | Yes | *(none — empty ⇒ "Invalid Report Type")* | Report style. One of `"SQL Server"`, `"Text File"`, `"Excel Spreadsheet"`, `"Per-Plate Html Files"`, `"Per-Plate Text Files"`. Evaluated as an expression. |
| `path` | expression-capable string | — | Yes | *(none — required for all styles)* | Output target. For file styles: full file path (Text/Excel) or directory (Per-Plate HTML/Text). For SQL Server: the **Data Source / server** (mirrors `server`). Evaluated as an expression. |
| `tableName` | expression-capable string | — | Conditional (SQL) | `""` | SQL table-name prefix. A `mm_dd_yyyy_hh_nn_ss` timestamp is appended directly at runtime (no leading `_`) so each run writes a fresh table. Must contain only letters, digits, and `_`. |
| `connectionString` | expression-capable string | — | Conditional (SQL) | `""` | ADO connection string for the SQL target. **A single leading `@` is silently stripped** before connecting (see Traps). Normally auto-built by the editor from `server`/`catalog`/`userName`/`password`/`authent`. |
| `catalog` | string | — | Conditional (SQL) | `""` | Database name → passed as `Initial Catalog` / `DB` on the SQL connection. **Not expression-evaluated** (read verbatim). |
| `userName` | string | — | No (editor state) | `""` | SQL login user. Read at enqueue but **not used there** — the editor folds it into `connectionString` on save. Persist it, but `connectionString` is what actually authenticates. |
| `password` | string | — | No (editor state) | `""` | SQL login password. Same status as `userName`: editor state, folded into `connectionString`; not consumed by enqueue directly. |
| `server` | string | — | No (editor state) | `""` | SQL server name. **Never read by the step** — editor-only; duplicated into `path` (Data Source) and `connectionString`. Inactive — not read at runtime. |
| `authent` | boolean | — | No (editor state) | `false` | "Use NT Authentication" (Windows integrated auth). **Never read by the step** — the editor uses it to build `connectionString` (`Integrated Security=SSPI` when `true`, else `User ID`/`Password`). Inactive — not read at runtime. |

The editor writes all nine keys on every save. File-style reports (Text/Per-Plate/Excel) ignore
every SQL-only key; author them anyway to match what the editor produces.

Universal base keys (`caption`, `dynamic?`, `disabled`, …) — see
[Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md). No item array.

## Enumerated / Constrained Values

### type (report style)

Matched case-insensitively against the values listed below.

| Value | Meaning |
|-------|---------|
| `"SQL Server"` | One database table (all plates), timestamp appended to the name. Uses `connectionString`/`tableName`/`catalog`. |
| `"Text File"` | Single delimited text file listing all plates. `path` = file path. |
| `"Excel Spreadsheet"` | Single `.xls` file. `path` = file path; filename must end `.xls`. **Caveat:** validation accepts it (directory / filename / `.xls` extension are checked) but the enqueue path has no Excel writer, so the step silently succeeds without producing any output. Do not author this style; it is also absent from the current manual's Report Style list. |
| `"Per-Plate Html Files"` | Directory of one HTML file per plate. `path` = directory. (Enqueue constant spells it `Per-Plate Html Files`; the editor combo/manual show `Per-Plate HTML Files` — matching is case-insensitive so both match.) |
| `"Per-Plate Text Files"` | Directory of one text file per plate. `path` = directory. |

## Cross-Field Validation Rules

1. Unrecognized `type` (code 1).
2. `Text File`: directory of `path` must exist (code 2); filename must be present and not a directory (code 5).
3. `Excel Spreadsheet`: directory must exist (code 3); filename present with `.xls` extension (code 4).
4. `Per-Plate Html Files`: `path` must be an existing directory (code 6).
5. `Per-Plate Text Files`: `path` must be an existing directory (code 7).
6. `SQL Server`: the connection must open (code 9); `tableName` must be non-empty (code 10); every `tableName` character must be a letter, digit, or `_` (code 8).
7. Any other configuration fault fails the step.

## Traps (author carefully)

- **`connectionString` leading `@` is silently stripped.** The `@` is the editor's marker for "Report-Location field holds a literal connection string". The runtime removes it before connecting. A literal connection string that genuinely starts with `@` will lose that character.
- **Windows-auth key is `authent`, not `ntAuth`.** The Create Data Set step uses
  `NTAuth`; this step persists `Authent`. `ntAuth` here is ignored.
- **`connectionString` wins over the individual SQL fields.** When authored directly, the
  step only additionally reads `catalog` (→ `Initial Catalog`) and `tableName`.

## Structural Context

Leaf step. Do not include `subSteps`.

## Canonical Examples

Text-file report (all nine keys present; SQL fields ignored for this style):

```json
{
  "stepType": "Data Set Reporting",
  "parameters": {
    "type": "Text File",
    "path": "C:\\BiomekReports\\run1.csv",
    "tableName": "",
    "connectionString": "",
    "catalog": "",
    "userName": "",
    "password": "",
    "server": "",
    "authent": false
  }
}
```

SQL Server report with Windows authentication:

```json
{
  "stepType": "Data Set Reporting",
  "parameters": {
    "type": "SQL Server",
    "path": "SQLHOST\\BIOMEK",
    "tableName": "RunResults",
    "connectionString": "Provider=SQLOLEDB.1;Integrated Security=SSPI;Initial Catalog=BiomekDB;Data Source=SQLHOST\\BIOMEK",
    "catalog": "BiomekDB",
    "userName": "",
    "password": "",
    "server": "SQLHOST\\BIOMEK",
    "authent": true
  }
}
```

## Common Mistakes

- **Giving a file path where the style expects a directory (or vice-versa).** `Per-Plate HTML Files`
  and `Per-Plate Text Files` write one file per plate into a **directory** — `path` must be an
  existing directory. `Text File` and `Excel Spreadsheet` need a full **file path** (Text/Excel's
  parent directory must exist and the path must evaluate to a filename). Passing a file to a Per-Plate style,
  or a directory to Text/Excel, hits the corresponding invalid-path/filename error.
- **Forgetting the `.xls` extension for Excel.** `Excel Spreadsheet` requires the filename to end in
  `.xls`, else the step fails at enqueue.
- **Using `NTAuth`/`ntAuth` casing instead of `authent`.** Windows-auth is represented as `authent` here
  (Create Data Set represents it as `NTAuth`). Anything else is silently ignored and the editor won't build
  Integrated-Security into `connectionString`.
- **Assuming `userName`/`password`/`server`/`authent` override `connectionString`.** They don't. At
  runtime, `connectionString` wins; the individual fields are editor-only state that the editor folds
  into `connectionString` on save. To change credentials at runtime, edit `connectionString`.
- **Leading `@` in `connectionString` disappears.** The `@` is the editor's "raw string" marker
  and is silently stripped before connecting. A literal connection string that genuinely starts
  with `@` will lose that character.
- **Case-mismatch between source-form and editor/manual form for Per-Plate HTML.** The enqueue
  constant spells it `Per-Plate Html Files`; the editor combo and manual show `Per-Plate HTML Files`.
  Matching is case-insensitive so either works; real exports and manual examples may show
  different casing.
- **Authoring only some of `authent`/`server`/`userName`/`password` without a `connectionString`.**
  Those fields are only wired through the editor; without a corresponding `connectionString` the
  runtime has nothing to connect with. Either author a full `connectionString`, or copy from an
  export where the editor built one.
