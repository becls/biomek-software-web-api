# Span-8 Transfer From File

| Property | Value |
|----------|-------|
| stepType | `"Span-8 Transfer From File"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | Span-8 pods only (i5, i7) |

---

## Behavior Summary

Span-8 Transfer From File is a **Span-8-only** step that reads a comma-delimited text file
(`.csv`/`.txt`) and builds one well-to-well aspirate/dispense per data row.

The step uses the file contents and configured source/destination items panels to generate a *pick list*. The pick list is five parallel lists of source position, source well, destination position, destination well and volume.

The pick list is pipetted using the Span-8 Transfer engine.

**What the file supplies vs. what the step supplies.** The file must always contain the
**source well** column; it *may* also carry source position, destination position,
destination well and volume. Each override/read boolean decides whether a piece of data comes
from the file or from the step's own `items` panels:

- `overrideSource` / `overrideDestination` — **`true` means "do NOT read from file; use the
  fixed position configured on the source/destination item"**; `false` means "read the
  position from the file column." (This is inverted from the intuitive reading — see the
  Enumerated Values note and Validation Rule 3.)
- `readDestWells` — `true` reads the destination well from a file column; `false` computes the
  next unused destination well from the destination item's `pattern`/marks.
- `readVolume` — `true` reads the volume from a file column; `false` uses the destination
  item's `volume`.
- `skipZero` — drops rows whose volume evaluates to 0.

Use it when a downstream process (a hit-picking / cherry-picking output, a normalization
table, a primer-plating list) already knows exactly which source well goes to which
destination well and how much — i.e. an arbitrary, data-driven set of individual transfers
that would be impractical to express as patterned Transfer/Combine steps.

---

## The file-row → transfer mapping

Each **line** of the file is either:

* a comment (first character `'`, an apostrophe — ignored)
* the header row (skipped when `fileHeader` is `true`)
* a data row that becomes **one aspirate + dispense**.
  * Fields are split on commas by the file parser.

Column indices (`sourceCol`, `sourceWellCol`, `destCol`,
`destWellCol`, `volumeCol`) are **1-based** and are read directly from the step's parameters.

Which columns must exist depends on the booleans:

| Datum | Read from file when | Column key | Column required (>0) when |
|-------|---------------------|------------|---------------------------|
| Source position | `overrideSource == false` | `sourceCol` | `overrideSource == false` |
| Source well | *always* | `sourceWellCol` | always |
| Destination position | `overrideDestination == false` | `destCol` | `overrideDestination == false` |
| Destination well | `readDestWells == true` | `destWellCol` | `readDestWells == true` |
| Volume | `readVolume == true` | `volumeCol` | `readVolume == true` |

Wells may be given as **numeric well numbers** (`23`, `32`) *or* **alphanumeric addresses**
(`A5`, `B3`); the same file may mix both. They are converted against the
resolved labware geometry.

Volume is parsed as a floating-point number; a negative or
unparseable volume raises a "bad volume at line N" error.

Positions may be **deck positions** (`P4`, `P7`) or **labware names** (`Source1`, `Dest`); every field is first run
through the expression engine, so cells may contain expressions.

**Worked example** (5 columns, header row present):

```
Source,SourceWell,Dest,DestWell,Volume
Primers,11,Dest,1,80
Primers,5,Dest,1,35
...
```

The matching configuration is: `fileHeader: true`, `overrideSource: false`,
`overrideDestination: false`, `readDestWells: true`, `readVolume: true`,
`sourceCol: 1`, `sourceWellCol: 2`, `destCol: 3`, `destWellCol: 4`, `volumeCol: 5`.
Row 2 aspirates 80 µL from labware `Primers` well 11 into `Dest` well 1;
Row 3 aspirates 35 µL from labware `Primers` well 5 into the same `Dest` well 1
(two separate transfers into the same destination well). Other typical shapes include a
5-column layout with numeric wells only (`SamplePlate,HitWell,Dest,DestWell,Vol`), and
mixed layouts where source wells are alphanumeric (`A5`, `B3`, …).

**When the destination well is *not* read from file** (`readDestWells: false`): the step
ignores `destWellCol` and instead walks the destination item's selection `pattern`, handing
out the next unused well each row; when the current destination panel's wells are exhausted
it advances to the next destination item, and raises a "not enough destination wells" error if all destinations run out.

---

## Parameters Reference Table

Serialized (first-letter-lowercase) casing is shown.

> Universal base keys (`caption`, `defaultCaption`, `disabled`, …) — see
> [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
>
> Envelope / `comArray` shape — see [Method JSON Structure](../Method-JSON-Structure.md).
>
> Pipetting keys (`prototype`/`autoSelectPrototype`/`overrideHeight`/`height`/`heightFrom`)
> — see [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md).

> Unlike the plain `Transfer` step and other span-8 pipetting steps (such as
> 15-Span-8-Aspirate / 16-Span-8-Dispense) some keys are not serialized for this
> step.
>
> **Not serialized by this step:** `type`/`Type`, `podType`, `stop`,
> `replicates`, `numberOfTips`, `unloadLocation`, `groupTransfer`, `dynamic?`.
> Do **not** author these keys for this step type.

### Step-Level Keys — Identity & Pod

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | string | — | Yes | `""` | Name of the Span-8 pod. Must resolve to a Span-8 pod; otherwise enqueue fails (Rule 1). This argument is used as a literal pod name. This step cannot use an expression here. |
| `span8` | boolean | — | Yes | `true` | **Always `true`.** Marks the step as Span-8. **Real, functional**.. |

### Step-Level Keys — File Source (the distinguishing feature)

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `destCol` | integer | — | Yes | `0` | 1-based file column for destination position. Used/validated only when `overrideDestination == false`. |
| `destWellCol` | integer | — | Yes | `0` | 1-based file column for destination well. Used/validated only when `readDestWells == true`. |
| `filename` | string (expression-capable) | — | Yes | `""` | Full path to the `.csv`/`.txt` data file. Evaluated with the expression engine, so `=`-expressions and variables are allowed. An empty filename fails validation. In a Validated Run the file is auto-registered. |
| `fileHeader` | boolean | — | Yes | `false` | `true` ⇒ the first data line is a header and is skipped. |
| `fileInfoCollapsed` | boolean | — | No | `false` | **Inert (UI-state).** Whether the File Properties panel is collapsed in the editor. No runtime effect. |
| `overrideDestination` | boolean | — | Yes | `false` | `true` = use the fixed destination item position for every row; `false` = read destination position from `destCol`. Same inversion as `overrideSource`. |
| `overrideSource` | boolean | — | Yes | `false` | **`true` = do NOT read the source position from the file; use the fixed position on the source item** (`items[0].position`). **`false` = read the source position from `sourceCol`.** (Inverted from intuition) |
| `readDestWells` | boolean | — | Yes | `false` | `true` = read destination well from `destWellCol`; `false` = compute the next unused destination well from the destination item's `pattern`/marks. |
| `readVolume` | boolean | — | Yes | `false` | `true` = read the transfer volume from `volumeCol`; `false` = use the destination item's `volume`. |
| `skipZero` | boolean | — | Yes | `false` | `true` = rows whose volume evaluates to 0 are dropped. |
| `sourceCol` | integer | — | Yes | `0` | 1-based file column for source position. Used/validated only when `overrideSource == false`. |
| `sourceWellCol` | integer | — | Yes | `0` | 1-based file column for source well. **Always required** (>0). |
| `volumeCol` | integer | — | Yes | `0` | 1-based file column for volume. Used/validated only when `readVolume == true`. |

### Step-Level Keys — Probe / Tip Selection

Semantics identical to the `Transfer` and Span-8 aspirate/dispense steps.

See [Probe & Mandrel Selection](../concept-guides/03-probe-and-mandrel-selection.md#tip-mode-keys-transfer-combine-span-8-transfer-from-file-span-8-serial-dilution).

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `mandrelExpression` | string | — | Conditional | `""` | Expression evaluating to the probe selection; required when `useExpression == true`. |
| `useDisposableTips` | boolean | — | Yes | `false` | One of the Span-8 tip-mode keys. When `true`, all non-fixed (disposable-tip) probes are used and `useProbes` is ignored. Do not also set `useFixedTips`. |
| `useExpression` | boolean | — | Yes | `false` | `true` ⇒ probe selection comes from `mandrelExpression` instead of `useProbes`. |
| `useFixedTips` | boolean | — | Yes | `false` | One of the Span-8 tip-mode keys. When `true`, only fixed-tip probes are used and `useProbes` is ignored. Do not also set `useDisposableTips`. |
| `useMandrelSelection` | boolean | — | Yes | `true` | **Inert (UI-state).** The engine does not read it; it persists the editor's selection mode. |
| `useProbes` | comArray (boolean[8]) | — | Yes | All `true` | Which of the 8 Span-8 probes are active. `{"_biomekType":"comArray","arraySubtype":"boolean","values":[…8…]}`. At least one `true` (Rule 11). |

### Step-Level Keys — Tip Handling

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `useCurrentTips` | boolean | — | Yes | `false` | `true` = reuse tips already on the pod; `false` = load fresh tips from `tipLocation`. |
| `tipLocation` | string | — | Conditional | `""` | Tip labware class / deck position to load when `useCurrentTips == false`. |
| `leaveTipsOn` | boolean | — | Yes | `false` | `true` = keep tips on the pod after the step; `false` = unload when finished. |
| `changeTipsBetweenSources` | boolean | — | No | `false` | Change/wash tips when the source position changes between transfers. |
| `changeTipsBetweenDests` | boolean | — | No | `true` | Change/wash tips when the destination changes. |
| `splitVolume` | boolean | — | Yes | `false` | Split large transfers into evenly divided aspirations, chunked to the smaller of the tip's usable capacity (net of blowout and air gap) and the syringe's usable per-stroke capacity (net of system air gap, blowout and trailing air gap, then adjusted for calibration) — not the raw syringe maximum. The single-dispense-per-draw restriction is enforced by the editor only: a method authored directly in JSON can set `splitVolume` to `true` alongside a repeat count above 1 and will not be rejected when the method runs. |
| `splitVolumeCleaning` | boolean | — | Yes | `false` | Clean tips between partial (split) transfers. Only meaningful when `splitVolume` and a between-transfer change/wash are on. |
| `useJIT` | boolean | — | Yes | `true` | When `true`, aspirate and dispense operations are bundled into a single Just-In-Time block to ensure no other actions occur in between the aspirate and dispense. |
| `repeats` | string (expression-capable) | count **or** µL | Yes | `"1"` | Configuration for multidispense behavior. If `repeatsByVolume == false`: number of dispenses per draw. If `true`: max µL to aspirate per draw for repeat-dispensing. Multidispensing is not a common usage and requires specialized pipetting techniques. This should typically be authored and set to `"1"`.|
| `repeatsByVolume` | boolean | — | Yes | `false` | Selects the meaning of `repeats` (see above). |

### Step-Level Keys — Wash

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `span8Wash` | boolean | — | Yes | `false` | Passive (flush) Span-8 wash between transfers. Mutually exclusive with `wash` in the editor. |
| `span8WashVolume` | string (expression-capable) | mL | Yes | `"2"` | Passive-wash volume (mL). |
| `span8WasteVolume` | string (expression-capable) | mL | Yes | `"1"` | Volume dispensed to waste before/after the passive wash (mL). |
| `wash` | boolean | — | Yes | `false` | Active wash (aspirate/dispense at a wash station) between transfers. |
| `solvent` | string | — | Yes | `"Water"` | Active-wash solvent name. |
| `washVolume` | string (expression-capable) | µL or % | Yes | `"110%"` | Active-wash volume; a trailing `%` means percent of the previously used capacity of the tip. |
| `washCycles` | string (expression-capable) | count | Yes | `"3"` | Number of active-wash cycles. |
| `autoSelectActiveWashTechnique` | boolean | — | Yes | `false` | Auto-select the active-wash technique. |
| `activeWashTechnique` | string | — | Yes | `""` | Named active-wash technique; ignored when `autoSelectActiveWashTechnique == true` or a custom technique is bound. |
| `customActiveWashTechnique` | object (`_biomekType:"technique"`) | — | No | *(absent)* | Inline custom wash technique. Present only when a custom technique is defined. Takes precedence over `activeWashTechnique`. |

### Step-Level Keys — UI State / Inactive

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `showTipHandlingDetails` | boolean | — | No | `true` | **Inert (UI-state).** Tip-handling panel expanded (stored as the negation of "collapsed"). |
| `showTransferDetails` | boolean | — | No | `true` | **Inert (UI-state).** Transfer-details panel expanded. |
| `wizard` | boolean | — | No | `false` | **Inactive — not read at runtime** for authored methods — wizard-created flag; hides the pod panel in the editor only. |
| `items` | array (object[]) | — | Yes | `[]` | The two pipetting panels: `items[0]` = source, `items[1]` = destination. See Item-Level Keys. Additional destination items are allowed only when neither `overrideDestination` nor `readDestWells` is set (extra dest panels for pattern-based multi-destination filling). |

### Item-Level Keys

`items` is the serialized form of the source/destination pipetting panels.
`items[0]` is the source, `items[1]` the (primary) destination, Additional destination items are allowed only when neither `overrideDestination` nor `readDestWells` is set (extra dest panels for pattern-based multi-destination filling).

These keys mirror the
`Transfer` item keys (see [Transfer](11-Transfer.md)) and drive the per-row
aspirate/dispense build at enqueue.

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `source` | boolean | — | Yes | `true`(item0)/`false`(item1) | Marks source vs destination panel. |
| `position` | string (expression-capable) | — | Yes | `""` | Fixed deck position / labware name. **Always required non-empty** on both `items[0]` and `items[1]` (Rule 2) — enqueue validates it regardless of `overrideSource`/`overrideDestination`. Its *value* is only used for pipetting when the corresponding `override*` is `true` (or as the destination when `readDestWells == false`); when the file supplies the position it is otherwise informational, but must still be non-empty. An empty position fails validation. |
| `labwareClass` | string | — | Yes | `""` | Labware class at this position. Empty triggers the same "step not configured" validation (Rule 2). Also used to size the generated well `pattern`. |
| `volume` | string (expression-capable) | µL | Conditional | `""` | Per-transfer volume for the destination item; used when `readVolume == false`. Ignored when volume is read from the file. |
| `liquidType` | string | — | Yes | `"Water"` | Liquid type for technique selection (source: e.g. `"Well Contents"`; dest: e.g. `"Tip Contents"` or a named liquid). |
| `prototype` | string | — | Required | `""` | Named pipetting technique; required when `autoSelectPrototype == false`. **Key must be present** on every item, but may be empty. |
| `autoSelectPrototype` | boolean | — | No | `false` | Auto-select the technique. |
| `customPrototype` | object (`_biomekType:"technique"`) | — | No | *(absent)* | Inline custom technique; overrides `prototype`. Copied through only when bound. |
| `overrideHeight` | boolean | — | Yes | `false` | Override the technique's tip height with `height`/`heightFrom`. |
| `height` | number/expression string | mm | Conditional | `0.0` | Tip-height offset when `overrideHeight == true`. |
| `heightFrom` | integer or string | — | Conditional | `0` | Height reference: `0`/`"Liquid"`, `1`/`"Bottom"`, `2`/`"Top"`. |
| `customHeight` | boolean | — | No | `false` | Encoding discriminator for `height`: `false` = numeric offset; `true` = text/custom height, which also implies `overrideHeight`. Use `false` with numeric heights. See [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#height-encoding-keys). |
| `pattern` | comArray (boolean[wells]) | — | Conditional | *(local)* | Well-selection mask for the **destination** item when `readDestWells == false` (drives the next-unused-well walk). Required (non-empty selection) in that mode or a "pattern not specified" error fires (Rule 9). |
| `selectionInfo` | comArray (boolean[wells]) or int[] | — | Conditional | mirrors `pattern` | Section/quadrant indices or a duplicate of `pattern`. **Keep it in sync with `pattern`** — on the **destination** item it must be present and correct whenever `readDestWells == false` and item-level `useExpression == false`; an absent or malformed value fails the step. |
| `sectionExpression` | string | — | Conditional | `""` | Section expression when `useExpression == true` (item-level). |
| `useExpression` | boolean | — | No | `false` | Item-level: use `sectionExpression` instead of `pattern`. |
| `localPattern` | boolean | — | No | `true` | `pattern` is stored inline. When `false`, `referencedPattern` names a global pattern. |
| `dataSetPattern` | boolean | — | No | `false` | Derive the selection from a data set (`dataSetCondition`/`dataSetConditionType`). |
| `referencedPattern` | string | — | Conditional | `""` | Named global pattern when `localPattern == false`; a missing/unknown name fails validation. |
| `dataSetConditionType` | string | — | No | `""` | Data-set condition operator (used with `dataSetPattern`). |
| `dataSetCondition` | string | — | No | `""` | Data-set condition value. |
| `wellsX` | integer | — | No | auto | Column count of the labware; used for pattern indexing / print layout. |
| `colsFirst` | boolean | — | No | `true` | Column-major fill order (left-to-right, then down) for pattern-based destination assignment. Complement of `rowsFirst`; the editor's `MarkInfoGrid` always serializes both. |
| `rowsFirst` | boolean | — | No | `false` | Row-major fill order (top-to-bottom, then right) for pattern-based destination assignment. **The Transfer From File step reads *this* key** (not `colsFirst`) to select row-major walking when `readDestWells == false`. Complement of `colsFirst`; author both together (`rowsFirst: true, colsFirst: false` for row-major; `rowsFirst: false, colsFirst: true` for column-major). |
| `startAtSelection` | boolean | — | No | `true` | Start from the beginning of the selection (vs. from the mark). |
| `startAtMark` | boolean | — | No | `false` | Start from the labware mark. |
| `setMark` | boolean | — | No | `false` | When `true`, the mark on the labware is updated after the transfer completes to the last well accessed. |

---

## Enumerated / Constrained Values

### `overrideSource` / `overrideDestination` (note the inversion)

| Value | Meaning |
|-------|---------|
| `false` | **Read the position from the file** (`sourceCol` / `destCol`). This is the common case; the editor checkbox "File specifies … position in column" is **checked**. |
| `true` | Do **not** read from file — use the single fixed position on the source/destination `item`. Editor checkbox **unchecked**. |

### `readDestWells` / `readVolume`

| Value | Meaning |
|-------|---------|
| `true` | Read destination well / volume from the file column (`destWellCol` / `volumeCol`). |
| `false` | Compute destination well from the destination item's pattern/marks / use the item's `volume`. |

### `repeatsByVolume`

| Value | `repeats` means |
|-------|-----------------|
| `false` | Number of dispenses per aspirate (draw). |
| `true` | Maximum µL to aspirate per draw (repeat-dispensing). |

Both boolean values are valid.

### `heightFrom` (per item)

| Value | Meaning |
|-------|---------|
| `0` / `"Liquid"` | From liquid surface. |
| `1` / `"Bottom"` | From well bottom. |
| `2` / `"Top"` | From well top. |

---

## Cross-Field Validation Rules

Each rule describes the enqueue-time behavior it enforces.

1. **Span-8 pod required.** `pod` must resolve and be a Span-8 pod. A missing pod or a non-Span-8 pod each fail with a distinct error.
2. **Both panels must be configured.** `items` must have ≥2 entries and both the source
   (`items[0]`) and destination (`items[1]`) must have non-empty `position` **and**
   `labwareClass`; otherwise the step is treated as not configured.
3. **File-column validity depends on the booleans**: each required column that is missing or ≤0 fails with a distinct "invalid column" error.
4. **Filename required.** Empty `filename` (after expression evaluation) fails validation; a path that cannot be opened fails at file-open time.
5. **Row parsing.** A data row with fewer fields than a referenced column fails to parse.
6. **Position / labware resolution** (per row): an empty source or destination position, an unknown position, or missing labware / missing section info each fail with a distinct message.
7. **Well validity.** A source or destination well outside `1..WellsX*WellsY` (numeric or
   after alphanumeric translation) fails validation.
8. **Volume validity.** When `readVolume == true`, a non-numeric or negative cell fails at that row.
9. **Pattern required for computed destinations.** When `readDestWells == false`, the
   destination item must provide a selection: an empty selection or an unknown
   named pattern each fail. When `readDestWells == true`, `pattern` is **still
   required**. It may be set to all `true` but it must exist.
10. **Destination capacity.** With computed destinations, running out of unused wells across
    all destination items fails.
11. **Probe selection**: `useProbes` must have ≥1 `true`,
    or when `useExpression == true`, `mandrelExpression` must be non-empty. All selected
    probes must share tip type and syringe size at runtime.
12. **Active vs. passive wash are mutually exclusive** in the editor (`wash` clears
    `span8Wash` and vice-versa); do not author both `true`.
13. **Transfer step validations.** Because the step hands the pipetting off to
    the ordinary Span-8 Transfer, that engine surfaces additional enqueue errors, including:
    a blank liquid type; an unknown named liquid; an unknown technique; `useCurrentTips: true`
    with no tips on the pod; a bad wash / waste / repeat count value; and an active wash whose
    solvent has no wash station.
14. **Do not use passive wash with disposable tips.** Passive wash may only be
    used with fixed tips. Only use active wash when disposable tips are loaded.

---

## Structural Context

Leaf step; no `subSteps`. Omit `subSteps`
or provide `[]`.

---

## Canonical Example

Reads a 5-column file (source position, source well, dest position, dest well, volume) with a
header row onto Span-8 `Pod1`, auto-selecting
techniques, loading fresh BC230 tips, changing tips between transfers.

> The pod name is per-instrument configuration — read it from the target instrument rather than
> copying the example's value.

```json
{
  "stepType": "Span-8 Transfer From File",
  "parameters": {
    "activeWashTechnique": "",
    "autoSelectActiveWashTechnique": false,
    "changeTipsBetweenDests": false,
    "changeTipsBetweenSources": true,
    "defaultCaption": "Transfer From File",
    "destCol": 3,
    "destWellCol": 4,
    "fileHeader": true,
    "fileInfoCollapsed": false,
    "filename": "C:\\Biomek\\Data\\transferfromfile.csv",
    "items": [
      {
        "autoSelectPrototype": true,
        "colsFirst": true,
        "customHeight": false,
        "dataSetPattern": false,
        "height": 0.0,
        "heightFrom": 0,
        "labwareClass": "BCFlat96",
        "liquidType": "Well Contents",
        "localPattern": true,
        "overrideHeight": false,
        "position": "P5",
        "pattern": {
          "_biomekType": "comArray",
          "arraySubtype": "boolean",
          "values": [true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true]
        },
        "prototype": "",
        "referencedPattern": "",
        "rowsFirst": false,
        "sectionExpression": "",
        "setMark": false,
        "source": true,
        "startAtMark": false,
        "startAtSelection": true,
        "useExpression": false,
        "volume": "",
        "wellsX": 12
      },
      {
        "autoSelectPrototype": true,
        "colsFirst": true,
        "customHeight": false,
        "dataSetPattern": false,
        "height": 0.0,
        "heightFrom": 0,
        "labwareClass": "BCFlat96",
        "liquidType": "Tip Contents",
        "localPattern": true,
        "overrideHeight": false,
        "pattern": {
          "_biomekType": "comArray",
          "arraySubtype": "boolean",
          "values": [true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true, true]
        },
        "position": "P9",
        "prototype": "",
        "referencedPattern": "",
        "rowsFirst": false,
        "sectionExpression": "",
        "setMark": false,
        "source": false,
        "startAtMark": false,
        "startAtSelection": true,
        "useExpression": false,
        "volume": "",
        "wellsX": 12
      }
    ],
    "leaveTipsOn": false,
    "mandrelExpression": "",
    "overrideDestination": false,
    "overrideSource": false,
    "pod": "Pod1",
    "readDestWells": true,
    "readVolume": true,
    "repeats": "1",
    "repeatsByVolume": false,
    "showTipHandlingDetails": true,
    "showTransferDetails": true,
    "skipZero": false,
    "solvent": "Water",
    "sourceCol": 1,
    "sourceWellCol": 2,
    "span8": true,
    "span8Wash": false,
    "span8WashVolume": "2",
    "span8WasteVolume": "1",
    "splitVolume": false,
    "splitVolumeCleaning": false,
    "tipLocation": "BC230",
    "useCurrentTips": false,
    "useDisposableTips": true,
    "useExpression": false,
    "useFixedTips": false,
    "useJIT": true,
    "useMandrelSelection": true,
    "useProbes": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true, true, true, true, true, true, true, true]
    },
    "volumeCol": 5,
    "wash": false,
    "washCycles": "3",
    "washVolume": "110%",
    "wizard": false
  }
}
```

## Common Mistakes

- **Missing `filename` or unreadable path**: the step reads `filename` at enqueue and opens the CSV file up-front; a bad path fails before any pipetting starts.
- **CSV column indices off-by-one**: `sourceCol`, `sourceWellCol`, `destCol`, `destWellCol`, `volumeCol` are **1-based**. Zero-based indices resolve to the wrong column silently — the transfer runs but reads incorrect data.
- **Header row mis-set**: setting `fileHeader: false` when the file has a header treats the header text as data; setting it `true` when the file has no header skips a real row. Verify against the actual file.
- **Mixing `overrideSource: true` with a non-zero `sourceCol`**: when `overrideSource` is `true` the file's source-position column is *not* read, so `sourceCol` is ignored. Leave `sourceCol: 0` (or reset it) when overriding, to avoid confusion for later readers. The same applies to `overrideDestination`/`destCol`, `readDestWells`/`destWellCol`, and `readVolume`/`volumeCol`.
- **Assuming labware auto-derives**: each item carries its own `labwareClass` (`items[0].labwareClass` for the source panel, `items[1].labwareClass` for the destination) and must match the deck positions the file's rows reference. On mismatch the runtime throws a class-mismatch error.
