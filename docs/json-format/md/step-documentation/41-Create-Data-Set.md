# Create Data Set

| Property | Value |
|----------|-------|
| stepType | `"Create Data Set"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | All (i5, i7, i3) — no hardware restriction |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Envelope / `_biomekType` / `comArray` / structural rules: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

Create Data Set reads a comma- or tab-delimited **text file** (which must reside on the local
controller) or a **SQL Server table** and attaches the resulting values as one or more **data sets**
to a piece of destination labware, one value per well. At enqueue time the step resolves the source
and destination labware on the deck, verifies neither is a Reservation or Lid, checks that the source
has no more wells than the destination, and then loads every row of the source file/table into a
temporary SQL Server table (`tempdb`, `127.0.0.1`, integrated security — for the file path) or opens
the user-specified database. It then builds a `#Platemap` temp table keyed on plate barcode + well
number, joins it to the source rows, and writes each requested column back into a newly created data
set on the destination labware. Rows are matched to wells in one of two modes: **by plate barcode /
well number** (`useWellNum: true`) or **by sample ID** — a join between a chosen file/table column and
an existing source data set (`useWellNum: false`). Each destination data set can be flagged
independently as *reportable* (included in reports) and *copy-on-transfer* (tracked during pipetting).
Use this step to bring externally-generated per-well data (sample IDs, concentrations, flags, etc.)
into a method so later steps (Transfer/Combine well-pattern-from-data-set, Data Set
Processing/Reporting, View Data Sets) can consume it.

> **Requires SQL Server.** Even the "read from file" path opens a local SQL Server connection
> (`Provider=SQLOLEDB;Data Source=127.0.0.1;Initial Catalog=tempdb;Integrated Security=SSPI`) and uses
> `BULK INSERT` to stage the file. The manual notes database tables are imported only from SQL Server
> 2014, and that source string columns must be `VARCHAR`, not `NCHAR`
> (see Biomek reference manual pages 23-8 and 23-10).

## Parameters Reference Table

### Step-Level Keys

All keys below are read at enqueue unless marked **inactive — not read at runtime**. Casing shown
is the serialized (camelCase) form; see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md#1-key-name-casing-spelling-global-rule) §1. Universal base keys (`caption`, `disabled`, `dynamic?`, …) are **not** listed —
see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md). The editor writes all 25 config keys plus the 3
`panelState*` keys on every save.

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `sourceFile` | string (expression-capable) | — | Conditional | `""` | Full path to the delimited text file to read. Required when `useDatabase` is `false`. Must exist on the local controller. Evaluated as an expression. |
| `sourcePos` | string (expression-capable) | — | Yes | `""` | Deck position of the **source** labware whose barcode/well or sample-ID data set is matched against the file/table. Evaluated as an expression; must resolve to a bound deck position. |
| `sourceDepth` | string (expression-capable integer) | stack index | No | `"0"` | Stack depth of the source labware (0 = top piece). Read as a string and evaluated as an integer expression. Editor default is `"0"`; at enqueue an absent key falls back to `0`. |
| `sourceDataSetName` | string (expression-capable) | — | Conditional | `""` | Name of the **existing** source data set that holds sample IDs. Used only when `useWellNum` is `false` (Match Sample ID mode). Evaluated as an expression. |
| `joinOnFieldName` | string (expression-capable) | — | Conditional | `""` | Column name in the file/table to join against `sourceDataSetName` when matching by sample ID (`useWellNum: false`). |
| `tableWellColName` | string (expression-capable) | — | Conditional | `""` | Column name in the file/table containing the **well number/ID**. Used when `useWellNum` is `true`. Also used to auto-detect alpha vs. numeric well IDs. |
| `tablePlateIDColName` | string (expression-capable) | — | No | `""` | Column name in the file/table containing the **plate barcode**. Optional even in well-number mode: if empty, all rows are assumed to apply to the plate (manual 23-14). |
| `destPos` | string (expression-capable) | — | Yes | `""` | Deck position of the **destination** labware on which the data sets are created. Evaluated as an expression; must resolve to a bound deck position. |
| `destDepth` | string (expression-capable integer) | stack index | No | `"0"` | Stack depth of the destination labware (0 = top). Same handling as `sourceDepth`. |
| `destDataSetNames` | string (comma-delimited list) | — | Yes | `""` | Comma-separated names of the data sets to create on the destination labware. Must yield ≥1 name. A name of `Volume` is skipped (the Volume data set is reserved/auto). |
| `destDataSetSources` | string (comma-delimited list) | — | Yes | `""` | Comma-separated file/table **column names**, positionally paired with `destDataSetNames`: the *n*-th column supplies the values for the *n*-th created data set. |
| `destDataSetReportable` | comArray (boolean[]) or boolean | — | No | `false` | Per-data-set "Include in Reports" flags, indexed to `destDataSetNames` order. When a boolean array, sets each new data set's `Reportable`. A scalar `false` means "not an array" → no per-set override applied. |
| `destDataSetCopyOnTransfer` | comArray (boolean[]) or boolean | — | No | `false` | Per-data-set "Track Data While Pipetting" flags, indexed to `destDataSetNames` order. When a boolean array, sets each data set's `CopyOnTransfer`. Scalar `false` = no override. |
| `firstRowHeader` | boolean | — | No | `false` | `true` = the first row read is a header row supplying column names; `false` = columns are auto-named `Column1`, `Column2`, … |
| `readStartNum` | integer | 1-based row | No | `1` (Enqueue) / `0` (editor initial) | Row number at which to begin reading the file. Enqueue default is `1`; the editor's spin control initializes to `0`. Combined with `firstRowHeader` to compute the first row. |
| `useWellNum` | boolean | — | No | `false` | Matching mode: `true` = match by plate barcode + well number (uses `tablePlateIDColName`/`tableWellColName`); `false` = match by sample ID (uses `joinOnFieldName`/`sourceDataSetName`). |
| `commaDelimited` | boolean | — | No | `true` | File field delimiter: `true` = comma-delimited, `false` = tab-delimited. Ignored when `useDatabase` is `true`. **Default is `true`.** |
| `useDatabase` | boolean | — | No | `false` | Source mode: `true` = read from a SQL Server table; `false` = read from `sourceFile`. |
| `databaseAddress` | string (expression-capable) | — | Conditional | `""` | SQL Server data source / server address. Used when `connectionString` is empty. |
| `databaseName` | string (expression-capable) | — | Conditional | `""` | SQL Server initial catalog (database name). Used when `connectionString` is empty. |
| `tableName` | string (expression-capable) | — | Conditional | `""` | Database table name to read (Match column data is `VARCHAR`). In file mode this key is **overwritten** internally to `#tmpTbl`; the authored value only matters for `useDatabase: true`. |
| `userName` | string | — | No | `"sa"` | SQL login user name. **Default is the literal `"sa"`.** Ignored when `ntAuth` is `true` or a `connectionString` is supplied. |
| `password` | string | — | No | `""` | SQL login password. Ignored when `ntAuth` is `true` or a `connectionString` is supplied. Stored in cleartext in the method. |
| `ntAuth` | boolean | — | No | `false` | `true` = connect with Windows Integrated Security (SSPI) instead of `userName`/`password`. (Serialized casing is `ntAuth`; source constant is `NTAuth`.) |
| `connectionString` | string | — | No | `""` | Raw ADO connection string. When non-empty it **overrides** `databaseAddress`/`databaseName`/`userName`/`password`/`ntAuth`. **Authoring note:** the editor strips a leading `@` when persisting this value, so the stored/JSON value is the connection string **without** the `@` prefix the editor uses as its "raw string" marker. |
| `panelStateFile` | boolean | — | No | `false` | **Inactive — not read at runtime.** Collapsed state of the *File Options* panel in the editor. |
| `panelStateInput` | boolean | — | No | `true` | **Inactive — not read at runtime.** Collapsed state of the *Input Options* panel (default collapsed). |
| `panelStateOutput` | boolean | — | No | `true` | **Inactive — not read at runtime.** Collapsed state of the *Output Options* panel (default collapsed). |

> **`alphaPosID` is not a dictionary key.** It is computed at enqueue from the
> data (alpha well IDs like `A1` vs. numeric), never read from or written to `parameters`. Do not
> author it.
>
> **`destDataSets` is never persisted** to the method JSON. Do not author it.

## Enumerated / Constrained Values

### Source mode (`useDatabase`)

| Value | Meaning |
|-------|---------|
| `false` | Read from the delimited text file at `sourceFile` (staged via `BULK INSERT` into `tempdb`). |
| `true` | Read from the SQL Server table `tableName` on `databaseAddress`/`databaseName`. |

### Matching mode (`useWellNum`)

| Value | Meaning |
|-------|---------|
| `true` | Match by **plate barcode + well number**. Uses `tablePlateIDColName` (optional) and `tableWellColName`. |
| `false` | Match by **sample ID**: join file/table column `joinOnFieldName` to existing source data set `sourceDataSetName`. |

### Delimiter (`commaDelimited`)

| Value | Meaning |
|-------|---------|
| `true` | Comma-delimited file (`FIELDTERMINATOR = ','`). Default. |
| `false` | Tab-delimited file (`FIELDTERMINATOR = '\t'`). |

### Authentication (`ntAuth`)

| Value | Meaning |
|-------|---------|
| `true` | Windows Integrated Security (`Integrated Security=SSPI`); `userName`/`password` ignored. |
| `false` | SQL login using `userName` (default `"sa"`) and `password`. |

## Cross-Field Validation Rules

1. **Source labware must exist.** `sourcePos` is expression-evaluated and must be a bound deck
   position, else the step fails at enqueue.
2. **Source labware type restriction.** The source may not be a Reservation or Lid.
3. **Destination labware must exist.** `destPos` must resolve to a bound deck position, else the step fails at enqueue.
4. **Destination labware type restriction.** The destination may not be a Reservation or Lid.
5. **Source no larger than destination.** Compared via the Volume data set sizes; the step fails if the source has more wells than the destination.
6. **At least one destination data set.** `destDataSetNames` must parse to ≥1 name. A name of
   `Volume` is silently skipped (reserved).
7. **File must exist (file mode).** When `useDatabase` is `false`, the step fails if the file is missing.
8. **File must have columns (file mode).** After header parsing, the step fails if no columns are found.
9. **Well column must exist in file (file + well-number mode).** The step fails if
   `tableWellColName` is not among the header columns.
10. **Table must be non-empty (database mode).** The step fails if the referenced table has no rows. Skipped when `connectionString` is non-empty (the raw-connection-string path exits before this check).
11. **Well column must exist in table (database + well-number mode).** The step fails if `tableWellColName` is not a
    column of the table. Also skipped when `connectionString` is non-empty.
12. **Stack depth must resolve to labware.** The step fails if no labware exists at the referenced deck position, or at the requested stack depth.
13. **Source-mode required keys.** `useDatabase: false` requires `sourceFile` (and honors `commaDelimited` / `firstRowHeader` / `readStartNum`). `useDatabase: true` requires `tableName` **plus** either a non-empty `connectionString` **or** `databaseAddress` + `databaseName` with credentials (`ntAuth: true`, or `userName`/`password`).
14. **Matching-mode required keys.** `useWellNum: true` requires `tableWellColName` (and optionally `tablePlateIDColName`); `useWellNum: false` requires `joinOnFieldName` **and** `sourceDataSetName`.
15. **Positional pairing.** Keep `destDataSetNames`, `destDataSetSources`, and (when arrays) `destDataSetReportable` / `destDataSetCopyOnTransfer` at equal length and order — the *n*-th entries are consumed together.

> **Positional pairing (not enqueue-guarded).** `destDataSetSources` (columns) and
> `destDataSetNames` (data-set names) are consumed by parallel index; a length mismatch is not
> caught by an explicit validation. If `destDataSetSources` has **fewer** entries than
> `destDataSetNames`, the step raises a list-index error at run time; if it has **more**, the extra
> column names are silently ignored. An order mismatch (equal lengths but wrong pairing) silently
> misassigns columns. `destDataSetReportable` / `destDataSetCopyOnTransfer`, when arrays, are also
> indexed positionally against `destDataSetNames`. Keep all four lists the same length and order.

## Structural Context

Leaf step. Do not include `subSteps` (omit, or an empty array).

## Canonical Example

File-based, match-by-sample-ID (creates one `SampleID` data set from a comma-delimited file):

```json
{
  "stepType": "Create Data Set",
  "parameters": {
    "commaDelimited": true,
    "connectionString": "",
    "databaseAddress": "",
    "databaseName": "",
    "destDataSetCopyOnTransfer": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true]
    },
    "destDataSetNames": "SampleID",
    "destDataSetReportable": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true]
    },
    "destDataSetSources": "Sample",
    "destDepth": "0",
    "destPos": "P5",
    "firstRowHeader": true,
    "joinOnFieldName": "Barcode",
    "ntAuth": false,
    "panelStateFile": false,
    "panelStateInput": true,
    "panelStateOutput": true,
    "password": "",
    "readStartNum": 1,
    "sourceDataSetName": "SampleID",
    "sourceDepth": "0",
    "sourceFile": "C:\\BiomekData\\samples.csv",
    "sourcePos": "P5",
    "tableName": "",
    "tablePlateIDColName": "",
    "tableWellColName": "",
    "useDatabase": false,
    "userName": "sa",
    "useWellNum": false
  }
}
```

## Common Mistakes

- **Confusing this step's `sourcePos` / `destPos` with `labwareAt` (used by 45 View Data Sets)**
  or with 42 Data Set Management's single-`position` key. Create Data Set has TWO deck positions — the source
  labware (barcode/well data joins against the file/table) and the destination labware (the
  new data sets are created here). Using `labwareAt` on this step is a no-op; enqueue then
  fails because the unbound `sourcePos` evaluates to an empty string.
- **Deck positions bound but no labware placed there at run-time**: `sourcePos` and `destPos`
  are validated at enqueue for a *bound* deck position. If an upstream Instrument Setup step
  hasn't placed labware at the referenced positions, the run halts with a no-labware-at-position error.
  Add an Instrument Setup upstream (or use runtime labware placement) so both `sourcePos` and `destPos`
  are populated when this step runs.
- **`destDataSetNames` and `destDataSetSources` length mismatch**: they are positionally
  paired — the *n*-th name is filled from the *n*-th column-source. A **shorter**
  `destDataSetSources` raises a list-index error when the step runs; a **longer** one
  silently ignores the extras. Keep both lists the same length and order.
- **`useDatabase: false` with `commaDelimited: true` but a tab-delimited file**: silently
  reads incorrect data — set `commaDelimited: false` for tab files.
