# Define Pattern

| Property | Value |
|----------|-------|
| stepType | `"Define Pattern"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | All |

## Behavior Summary

The Define Pattern step creates a named well-selection pattern that can be referenced by other steps (e.g., pipetting steps to limit operations to specific wells). The pattern is a boolean array where each element corresponds to a well in the labware. The pattern can be defined manually (a fixed array of booleans), read from a CSV file at runtime, or prompted from the user interactively. The pattern is stored globally and persists for the duration of the method run.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `name` | `expression-capable string` | — | Yes | — | Name of the pattern being defined. Must not be empty. |
| `type` | `expression-capable string` | — | Yes | — | Labware class name (defines well geometry). Must match an installed labware class **exactly** (examples: `BCFlat96`, `Arrowhead_AbGene_96_PCR`, `Labcyte_384PPL`). |
| `readFromFile` | `boolean` | — | No | `false` | If `true`, read pattern from a CSV file. |
| `promptForPattern` | `boolean` | — | No | `false` | If `true`, show a UI prompt at runtime. |
| `file` | `expression-capable string` | — | Conditional | — | Path to CSV file. Required when `readFromFile` is `true`. |
| `headerRow` | `boolean` | — | Conditional | `false` | If `true`, first row of CSV is treated as a header. Used with `readFromFile`. |
| `matchColumn` | `integer` | — | Conditional | `0` | 0-based column index in CSV for matching. Used with `readFromFile`. |
| `wellColumn` | `integer` | — | Conditional | `0` | 0-based column index in CSV for well identifiers. Used with `readFromFile`. |
| `useAllLines` | `boolean` | — | Conditional | `false` | If `true`, all rows in CSV contribute to pattern. If `false`, matching lines only are used. Used with `readFromFile`. |
| `matchBarcode` | `boolean` | — | Conditional | `true` | If `true`, the property and position defined in `matchProperty` and `matchPosition` are used to determine which lines in the CSV contribute to the pattern. If `false`, the expression defined in `matchExpression` is used to determine which lines contribute to the pattern. Used with `readFromFile` when `useAllLines` is `false`. |
| `matchPosition` | `expression-capable string` | — | Conditional | — | Deck position of the labware whose property will be read for matching. Used with `readFromFile` when `matchBarcode` is `true` and `useAllLines` is `false`. |
| `matchProperty` | `expression-capable string` | — | Conditional | `"Barcode"` | Name of the property to read from the labware at `matchPosition`. Used with `readFromFile` when `matchBarcode` is `true` and `useAllLines` is `false`. |
| `matchExpression` | `expression-capable string` | — | Conditional | — | Value (or `==`-prefixed case-insensitive regex) compared against each CSV row's match column to determine which rows in the CSV contribute to the pattern. Used with `readFromFile` when `matchBarcode` is `false`. |
| `pattern` | comArray of booleans (`<comArray:boolean:N>`) | — | Conditional | — | Manual well-selection array. Required when `readFromFile` is `false` and `promptForPattern` is `false`. **Serializes as a typed comArray** (`<comArray:boolean:96>` or `<comArray:boolean:384>`), **not** a plain JSON boolean array; see [Method JSON Structure](../Method-JSON-Structure.md) for comArray shape. |
| `sectionExpression` | `expression-capable string` | — | No | `""` | Expression-based pattern (advanced); Used only when `useExpression` is `true` and `readFromFile` is `false`. |
| `useExpression` | `boolean` | — | No | `false` | If `true`, evaluates `sectionExpression` instead of the manual `pattern`, when `readFromFile` is `false`. |
| `selectionInfo` | comArray of booleans (`<comArray:boolean:N>`) | — | No | — | Parallel copy of the well-selection array, written alongside `pattern`. Inactive at enqueue — not read at runtime (the enqueue read is `pattern`), but the editor always emits it. **Always author this key alongside `pattern` for a manual pattern** (`readFromFile=false` and `useExpression=false`). |
| `wellsX` | `integer` | — | No | `1` | Wells-per-row of the referenced labware class. Always emitted by the editor. **Always author this key for a manual pattern** (`readFromFile=false` and `useExpression=false`). |
| `wellsAreNumbers` | `boolean` | — | No | — | **Obsolete and inactive — not read at runtime.** Preserve if present in an existing method; do not author from scratch. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._
>
> **On always-present optional keys.** The editor emits `readFromFile`, `promptForPattern`, `useExpression`, `sectionExpression`, `file`, `headerRow`, `matchColumn`, `wellColumn`, `useAllLines`, `matchBarcode`, `matchProperty`, `matchExpression`, `matchPosition`, `selectionInfo`, `wellsX`, and `pattern` on every editor-produced step, but enqueue reads them all tolerantly with defaults — except `pattern`, which is required in the manual branch (see its row).

## Enumerated / Constrained Values

None.

## Cross-Field Validation Rules

1. `name` **must not** be empty.
2. `type` **must** name a labware class defined in the project.
3. When `readFromFile` is `true`, `file` must name an existing CSV file.
4. When `readFromFile` is `true`, the CSV must have valid `matchColumn` and `wellColumn` indices.
5. Well identifiers in CSV must be valid for the labware type (numeric or alphanumeric A1 format).
6. When `readFromFile` is `false` and `promptForPattern` is `false`, `pattern` must be provided.
7. **CSV matching mode** (only when `readFromFile` is `true` and `useAllLines` is `false`) — `matchBarcode`, `matchPosition`, `matchProperty`, and `matchExpression` together pick which CSV rows apply:
   - **`matchBarcode = true`** (default): the step finds the labware at `matchPosition`, reads its `matchProperty` (default `"Barcode"`), and uses that value as the string each CSV row's match column is compared against. `matchPosition` is effectively required in this branch (an empty `matchPosition` will fail to resolve a labware); `matchExpression` is **not** used.
   - **`matchBarcode = false`**: `matchExpression` is used directly as the comparison target. If `matchExpression` starts with `==`, the remainder is treated as a case-insensitive regex; otherwise it is used as a literal string (after expression evaluation). `matchPosition` and `matchProperty` are **not** used in this branch.
   - When `useAllLines` is `true`, no matching is performed and all rows contribute regardless of `matchBarcode`/`matchPosition`/`matchProperty`/`matchExpression`.
8. `promptForPattern` is **not** a third pattern source — it is orthogonal to `readFromFile`. When `promptForPattern` is `true`, the runtime prompt is a *file* picker if `readFromFile` is `true` and a well-selection grid otherwise, and the prompted value supersedes the authored `file` / `pattern` for that run.

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

Manual pattern (wells 1–8 selected in a 96-well plate). Always author `wellsX` and `selectionInfo` alongside `pattern`.

```json
{
  "stepType": "Define Pattern",
  "parameters": {
    "name": "Column1Only",
    "type": "BCFlat96",
    "wellsX": 12,
    "pattern": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true,true,true,true,true,true,true,true,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false]
    },
    "selectionInfo": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true,true,true,true,true,true,true,true,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false,false]
    }
  }
}
```

Pattern from CSV file:

```json
{
  "stepType": "Define Pattern",
  "parameters": {
    "name": "SamplePattern",
    "type": "BCFlat96",
    "readFromFile": true,
    "file": "C:\\Methods\\patterns\\sample_list.csv",
    "headerRow": true,
    "matchColumn": 0,
    "wellColumn": 1,
    "matchBarcode": true,
    "matchPosition": "P1",
    "matchProperty": "Barcode"
  }
}
```

## Common Mistakes

- **Wrong `pattern` array length**: Must match the well count of the labware class (e.g., 96 booleans for a 96-well plate). Not statically enforced; fails at enqueue.
- **Serializing `pattern` as a plain JSON boolean array**: it must be a typed comArray (`<comArray:boolean:96>` / `:384>`). A raw `[true, false, …]` will not round-trip. See the Parameters table entry and [Method JSON Structure](../Method-JSON-Structure.md).
- **Using `promptForPattern` in automated contexts**: runtime prompts require a GUI and are unsuitable for unattended runs.
- **Authoring `wellsAreNumbers`**: inactive — not read at runtime; preserve if present on an existing method, do not add it to new steps.
