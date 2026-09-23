# Worklist

| Property | Value |
|----------|-------|
| stepType | `"Worklist"` |
| Category | Free-form container |
| Terminator | `"End"` (editor caption "End Worklist" is display text only — the step type is `"End"`) |
| Compatible Hardware | All |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Structure/envelope: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Worklist step reads a CSV worklist file and repeats its child steps once per data
row, binding that row's columns as scoped variables for the iteration. It combines the
iteration of a `Loop` with the per-iteration variable scoping of a `Let`, driven by the
rows of a file. At enqueue it parses the file, then for each data
row in the selected range it binds that row's `column → value` pairs and enqueues the
child steps inside that scope — the same scoping mechanism a `Let` step provides. Every
column value is bound as a **string**. If `loopVariable` is set, that row's 1-based
data-row number is also bound under that name, as a number. Iterate the whole file
(`whole?` = true) or a `start`..`end` data-row range. A child step reads a column by
writing `=ColumnName` in any expression-capable field (a column named `AmountA` is read
as `"=AmountA"`); the `loopVariable` name is referenced the same way. The step also
registers the file as a run attachment on validated runs.

## Parameters Reference Table

### Step-Level Keys

Every editor-produced step carries all five keys — emit all five, using `""` for the ones you
are not using.

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `filename` | `expression-capable string` | — | Yes | `""` *(constructor)* | Path to the worklist CSV file. Evaluated as an expression at run time, so an `=`-prefixed value is resolved before the file is opened. If the expression cannot be resolved, the run fails with the expression-evaluation error for that value. |
| `loopVariable` | `string` | — | No | `""` | Name of an extra variable bound in each iteration to that iteration's 1-based **data-row** number. With a partial range this is the absolute row number, not a counter restarting at 1 (with `start: "3"` the first iteration binds `3`). Bound as a number, not a string. Must **not** match any column name in the file (see rule 1). May be empty — treat it as genuinely optional. |
| `start` | `expression-capable string` | data row # (1-based) | Conditional | `""` | First data row to process. Read/validated only when `whole?` is false. Write it as a string (`"2"`, `"=(Col*8)-7"`), not a JSON number. |
| `end` | `expression-capable string` | data row # (1-based) | Conditional | `""` | Last data row to process. Read/validated only when `whole?` is false. Write it as a string (`"5"`, `"=(Col*8)"`), not a JSON number. |
| `whole?` | `boolean` | — | No | `true` | When true, iterate the entire worklist and ignore `start`/`end`. When false, iterate the `start`..`end` data-row range. Enqueue reads it tolerantly, so omission is safe. Note the trailing `?` is preserved in the serialized key. |

Universal base keys (`caption`, `isPreconfigured`, `dynamic?`, …) — see
[Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).

> **Do not author `let` / `weak` / `prompt` on a Worklist node.** Unlike `Let` and the
> other scope steps, Worklist does not persist these keys — its per-row scope is created
> internally, one binding set per data row. See
> [Base Keys & Serialization Conventions §4](00-Base-Keys-and-Conventions.md#4-keys-that-look-shared-but-are-genuinely-per-step).

## Enumerated / Constrained Values

No enumerated string values. `whole?` is a plain boolean; `start`/`end` are strings that
evaluate to data-row numbers.

### Worklist file format

- Comma-separated values. The **first** non-comment line is the header row; its fields
  become the column/variable names. A blank first line fails parsing.
- Every line after the header that is neither blank nor a comment is a **data row**.
  Row numbering for `start`/`end` and for `loopVariable` counts data rows only: the
  header, `#` comment lines, and blank lines are not counted, and the first data row is
  row 1.
- Lines whose first non-whitespace character is `#` are comments and are skipped.
- Lines must end with CRLF (`\r\n`) — a Windows CSV. A file saved with LF-only line
  endings is read as a single line: the whole file becomes the header row, so no data rows
  are found and the step iterates zero times. If the merged header contains an empty field,
  the run fails on the empty-variable-name check; otherwise it completes with no error.
- Fields may be double-quoted (`"..."`); surrounding quotes and surrounding whitespace
  are stripped.
- A quoted field may contain commas — a comma inside quotes does not split the field.
- An unclosed double quote fails parsing.
- Every column value is stored as a **string** in the per-row bindings.
- Each data row must have exactly as many values as there are header columns — otherwise
  parsing fails.
- A file with a header but no data rows is not an error: the step simply runs zero
  iterations.

Example worklist file:

```
Plates, AmountA
Plate1, 10
Plate3, 5
```

Two data rows. Iteration 1 binds `Plates` = `"Plate1"` and `AmountA` = `"10"` (plus the
`loopVariable`, if set, to `1`); iteration 2 binds `Plates` = `"Plate3"`, `AmountA` =
`"5"`, counter `2`. A child step reads them as `"=Plates"` and `"=AmountA"` in any
expression-capable field.

## Cross-Field Validation Rules

1. `loopVariable` **must not** equal any column name in the file.
2. When `whole?` is false, `start` **must** be non-empty.
3. When `whole?` is false, `end` **must** be non-empty.
4. `start` must be within 1..(number of data rows).
5. `end` must be within 1..(number of data rows).
6. `start` **must not** exceed `end`.
7. `filename` must resolve to a readable file: an empty path or a path that cannot be opened fails at enqueue.
8. The file must have a header row — no columns is an error, and an empty column name is an error.
9. Each data row must have the same field count as the header.
10. A child step that fails during a pass is re-raised (N is the 1-based data-row number).

## Structural Context

The Worklist step is a **free-form container**. It requires one child:

- **Trailing anchor (index 0)**: `"End"` with caption `"End Worklist"`.

**Constraints** (same shape as `Loop`/`Group`, per Method-JSON-Structure.md):

- `subSteps` must be present and end with the `"End"` terminator as the last element.
- Any number of operational steps may precede the terminator.
- A Worklist containing only `"End"` is structurally valid but iterates over no child work.

**Worklist does not participate in the `Break` protocol.** Like `If`, `Group`, and `Let`,
it does not consume a `Break` level — only `Loop` does. A `Break` inside a Worklist aborts the
current pass and propagates outward as an error rather than ending the row iteration
cleanly: with an enclosing `Loop`, that Loop consumes the level and terminates (which
ends the Worklist with it); with no enclosing `Loop`, the method fails at run time. To end a Worklist early, guard the child steps with an
`If` instead.

## Canonical Example

Iterate the whole worklist shown above, binding the data-row number as `Row`; the child
`Pause` reads the `Plates` column of the current row:

```json
{
  "stepType": "Worklist",
  "parameters": {
    "filename": "C:\\Methods\\WorkListInput.txt",
    "loopVariable": "Row",
    "start": "",
    "end": "",
    "whole?": true
  },
  "subSteps": [
    { "stepType": "Pause", "parameters": { "mode": "PromptedGlobal", "message": "=Plates" } },
    { "stepType": "Transfer", "parameters": {} },
    { "stepType": "End", "parameters": { "caption": "End Worklist" } }
  ]
}
```

Iterate only data rows 2–5:

```json
{
  "stepType": "Worklist",
  "parameters": {
    "filename": "=WorklistPath",
    "loopVariable": "Row",
    "start": "2",
    "end": "5",
    "whole?": false
  },
  "subSteps": [
    { "stepType": "Move Labware", "parameters": {} },
    { "stepType": "End", "parameters": { "caption": "End Worklist" } }
  ]
}
```

## Common Mistakes

- **`loopVariable` collides with a column name**: fails at enqueue. Pick a name not used as a CSV header.
- **Setting `start`/`end` but leaving `whole?` true**: the range is silently ignored. Set `whole?` to `false` to activate the range.
- **Assuming numeric CSV values stay numeric**: every value read from a *column* is bound as a **string**. Convert in the child step if arithmetic is needed. The `loopVariable` row number is the exception — it is bound as a number.
- **Off-by-one on rows**: `start`, `end`, and the `loopVariable` value are 1-based **data-row** numbers — the header line, `#` comment lines, and blank lines are not counted, and the first data row is row 1.
- **Emitting `start`/`end` as JSON numbers**: write `"2"`, not `2`. The editor writes both as strings and expression evaluation depends on the string form.
- **Confusing `end` (last data row) with `"End"` (terminator step)**: `end` is a parameter inside `parameters`; `"End"` is the step type that must be the last element of `subSteps`.
- **Writing `"stepType": "End Worklist"`**: "End Worklist" is only the editor caption. The step type is `"End"`.
- **Saving the worklist with Unix (LF) line endings**: the parser expects CRLF. An LF-only file yields no data rows and the step silently does nothing.
- **Dropping the trailing `?` from `whole?`**: the key is literally `whole?` in the serialized form — a `whole` key will not be read.
- **Expecting `Break` to end the Worklist**: it does not — see §Structural Context. Guard the child steps with an `If` instead.
