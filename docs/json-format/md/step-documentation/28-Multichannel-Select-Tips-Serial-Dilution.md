# Multichannel Select Tips Serial Dilution

> **Reference manual:** Chapter 20 documents these as *Select Tips* steps (the manual omits the *Multichannel* prefix); the JSON `stepType` keeps the `Multichannel Select Tips …` name.

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Select Tips Serial Dilution"` |
| Category | Leaf (does not accept child steps) |
| Terminator | N/A |
| Compatible Hardware | Multichannel pod only |

## Behavior Summary

The Select Tips Serial Dilution step performs a complete serial dilution
protocol using the multichannel pod's currently loaded tips. It orchestrates
multiple aspirate/dispense operations internally: optionally transferring
diluent to the plate, optionally transferring source compound from a master
plate, then performing sequential dilution transfers across rows or columns. It
can optionally wash tips between operations and discard excess volume at the
end.

The Select Tips Serial Dilution step can perform dilution across rows or across
columns, depending which arrangement of tips is loaded on the pod. It requires
equally spaced columns or rows of tips to be loaded.

## Parameters Reference Table

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

### Decision Flags

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `addDiluent?` | `boolean` | — | No | `false` | Include diluent transfer to plate before dilution. |
| `getSourceCompound?` | `boolean` | — | No | `false` | Transfer source from master plate to first well set. |
| `discardExcess?` | `boolean` | — | No | `false` | Discard excess volume from last well set. |
| `diluteFirstWells?` | `boolean` | — | No | `false` | When `addDiluent?` is `true`, also add diluent to the first row/column (otherwise the first row/column is left undiluted as the concentrated reference). Has no effect when `addDiluent?` is `false`. |
| `washAfterDiluent?` | `boolean` | — | No | `false` | Wash tips after diluent transfer. |
| `washAfterSource?` | `boolean` | — | No | `false` | Wash tips after master/source transfer. |
| `washWhenDone?` | `boolean` | — | No | `false` | Wash tips when serial dilution is complete. |

### UI Display Flags

All four are **editor-persistence only; not read at runtime**.

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `showTips?` | `boolean` | — | No | `false` | Show tips panel in UI. Editor-only. |
| `showPlate?` | `boolean` | — | No | `true` | Show dilution plate panel in UI. Editor-only. |
| `showDiluent?` | `boolean` | — | No | `true` | Show diluent panel in UI. Editor-only. |
| `showMaster?` | `boolean` | — | No | `false` | Show master plate panel in UI. Editor-only. |

### Wash Configuration (used when any `washAfter*?` / `washWhenDone?` flag is `true`)

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `activeWashTechnique` | `string` | — | Conditional | `""` | Name of the active-wash technique to apply between operations. Required when `autoSelectActiveWashTechnique` is `false` and any wash decision flag (`washAfterDiluent?`, `washAfterSource?`, or `washWhenDone?`) is `true`. |
| `autoSelectActiveWashTechnique` | `boolean` | — | No | `false` | If `true`, auto-select the wash technique instead of using the named `activeWashTechnique`. |
| `washCycles` | `expression-capable string` | count | Conditional | `""` | Number of wash cycles per wash operation. Required when any wash decision flag is `true`. |
| `washLiquid` | `string` | — | Conditional | `""` | Wash-solvent name (e.g. `"Water"`). Required when any wash decision flag is `true`. |
| `washVolume` | `expression-capable string` | µL or % | Conditional | `""` | Wash volume per cycle (e.g. `"110%"` or an absolute µL string). Required when any wash decision flag is `true`. |
| `customActiveWashTechnique` | object or `null` | — | No | `null` | Inline wash technique object (embedded `_biomekType: "technique"`). Populated by the technique editor; for JSON authoring, prefer `activeWashTechnique` by name or `autoSelectActiveWashTechnique: true`. Preserve on round-trip if present. |

### Plate Configurations (nested dictionaries)

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `basePlate` | `object` | — | Yes | — | Dilution plate configuration (see Plate Dictionary below). |
| `diluentPlate` | `object` | — | Conditional | — | Diluent source configuration. Required when `addDiluent?` is `true`. |
| `masterPlate` | `object` | — | Conditional | — | Master/source configuration. Required when `getSourceCompound?` is `true`. |

### Plate Dictionary Structure

The three plate dictionaries have **different** key sets — they are structured differently and do not share a common shape. All three share the following common keys:

| Key | Type | Required | Default | Description |
|-----|------|----------|---------|-------------|
| `location` | `string` | Yes | — | Deck position name or labware **instance** name. Expression-capable. |
| `volume` | `string` | Conditional | — | Transfer volume (expression-capable). Required on `basePlate` and `masterPlate`; on `diluentPlate` this key is UI-persisted but not read at enqueue — the diluent transfer volume is computed from `basePlate.volume` and `dilutionRatio`. |
| `labwareClass` | `string` | Yes | — | Labware class name. |
| `liquidtype` | `string` | Conditional | — | Liquid classification for technique selection. |
| `usingPlate?` | `boolean` | No | match labware type | Selects how the diluent/master plate is rendered in the printed/summary method — titer-plate (well column/row) vs. reservoir (section). **Keep it in sync with the plate's actual labware type.** It does **not** gate whether the plate participates (that is `addDiluent?` / `getSourceCompound?`), and it is not used for the base plate. |
| `autoSelectPrototype` | `boolean` | No | `false` | Per-plate technique auto-select (see [Pipetting Techniques](../Pipetting-Techniques-and-Templates.md)). |
| `prototype` | `string` | Conditional | `""` | Named pipetting technique for this plate. Required when `autoSelectPrototype` is `false` and no inline `customPrototype` is set. |
| `customPrototype` | object | Conditional | (unset) | Inline pipetting-technique object. Alternative to `prototype`/`autoSelectPrototype`; populated by the editor. |
| `operation` | `string` | No | (set by editor) | Editor-persistence only; written by the technique selector for its own filtering. Not read at runtime. Preserve on round-trip if present. |
| `overrideHeight` | `boolean` | No | `false` | Per-plate tip-height override — `true` ⇒ use `height`/`heightFrom` instead of the technique default. |
| `height` | `number` | Conditional | — | Tip-height offset (mm); used when `overrideHeight` is `true`. |
| `heightFrom` | integer or string | Conditional | — | Height reference: `0`=Liquid, `1`=Bottom, `2`=Top. Also accepts string aliases `"Liquid"`, `"Bottom"`, `"Top"`, matching the Multichannel Aspirate/Dispense/Mix and Select Tips Aspirate steps. Used when `overrideHeight` is `true`. |
| `customHeight` | `boolean` | No | `false` | Encoding discriminator for `height`: `false` = numeric offset; `true` = text/custom height, which also implies `overrideHeight`. Use `false` with numeric heights and carry the override on `overrideHeight`/`height`/`heightFrom`. Preserve on round-trip if present. See [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#height-encoding-keys). |

> _Each plate dictionary **must** have a pipetting technique configured — at least one of `autoSelectPrototype: true`, a named `prototype`, or an inline `customPrototype` must be present. Otherwise enqueue raises a no-technique error.

> _Each plate dictionary carries the **full per-plate pipetting + height sub-config** above, in addition
> to its geometry keys below. These behave exactly as the
> like-named keys on the Select Tips Multichannel Aspirate-Dispense steps._

**`basePlate`** (dilution plate) adds *only* the following keys — it does not
require `columnOffset`, `rowOffset`, or `reservoirSection`:

| Key | Type | Description |
|-----|------|-------------|
| `startWells` | `string` | Starting well index for dilution range to position the back left tip over |
| `endWells` | `string` | Ending well index for dilution range to position the back left tip over |
| `wellSet` | `string` | Offset rows/columns to dilute, when mandrel spacing does not match well spacing. For example, if performing a column dilution with a 96 head in a 384-well plate, a value of `1` hits rows 1, 3, etc., while `2` hits rows 2, 4, etc. |
| `discardExcessPosition` | `string` | Position for excess discard |

**`diluentPlate`** (diluent source) adds *only* the following keys — it has **no** `startWells`, `endWells`, or `wellSet`:

| Key | Type | Description |
|-----|------|-------------|
| `columnOffset` | `string` | Starting column (1-based, expression-capable) to position the back left tip over |
| `rowOffset` | `string` | Starting row (1-based, expression-capable) to position the back left tip over |
| `reservoirSection` | `string` | Reservoir section (for reservoir labware) to position the back left tip over |
| `dilutionRatio` | `expression-capable string` (real ≥ 1.0) | Dilution ratio (e.g., `"2.0"` for 1:2 dilution). Enqueue rejects values < 1.0. |

**`masterPlate`** (master/source) adds *only* the following keys — it has **no** `startWells`, `endWells`, `wellSet`, or `dilutionRatio`:

| Key | Type | Description |
|-----|------|-------------|
| `columnOffset` | `string` | Starting column (1-based, expression-capable) to position the back left tip over |
| `rowOffset` | `string` | Starting row (1-based, expression-capable) to position the back left tip over |
| `reservoirSection` | `string` | Reservoir section (for reservoir labware) to position the back left tip over |

## Cross-Field Validation Rules

1. This step **must** be inside a `"Multichannel Select Tips"` container.
2. The loaded tip pattern **must** be equally spaced.
3. Tips must already be loaded on the pod, or a pod-no-tips error is raised.
4. When `addDiluent?` is `true`, `diluentPlate` must be provided with valid `location` and `volume`.
5. When `getSourceCompound?` is `true`, `masterPlate` must be provided.
6. When `discardExcess?` is `true`, `basePlate.discardExcessPosition` must be set.
7. `diluentPlate.dilutionRatio` **must** evaluate to a real ≥ 1.0; enqueue rejects values below 1.0.
8. Each active plate dictionary (`basePlate` always; `diluentPlate` when `addDiluent?` is `true`; `masterPlate` when `getSourceCompound?` is `true`) **must** have a pipetting technique configured — either `autoSelectPrototype: true`, a named `prototype`, or an inline `customPrototype`. Otherwise enqueue raises a no-technique error.

## Structural Context

This step is a leaf — it goes inside a `"Multichannel Select Tips"` container but does not have its own subSteps.

## Canonical Examples

Simple serial dilution (1:2 across columns):

```json
{
  "stepType": "Multichannel Select Tips Serial Dilution",
  "parameters": {
    "addDiluent?": true,
    "getSourceCompound?": true,
    "discardExcess?": true,
    "washAfterSource?": true,
    "basePlate": {
      "location": "P4",
      "volume": "100",
      "labwareClass": "BCFlat96",
      "liquidtype": "Well Contents",
      "autoSelectPrototype": true,
      "startWells": "1",
      "endWells": "12",
      "wellSet": "1",
      "discardExcessPosition": "AW1"
    },
    "diluentPlate": {
      "location": "P3",
      "volume": "100",
      "labwareClass": "Reservoir",
      "liquidtype": "Well Contents",
      "autoSelectPrototype": true,
      "reservoirSection": "1",
      "dilutionRatio": "2.0"
    },
    "masterPlate": {
      "location": "P5",
      "volume": "100",
      "labwareClass": "BCFlat96",
      "liquidtype": "Well Contents",
      "autoSelectPrototype": true,
      "columnOffset": "1",
      "rowOffset": "1"
    }
  }
}
```

## Common Mistakes

- **Uneven tip pattern loaded**: unequally-spaced rows/columns fail — the pattern loaded by the preceding Select Tips Load must be a regular row or column pitch (see rule 2).
- **Enabling a decision flag without its plate/config**: `addDiluent?` needs `diluentPlate`; `getSourceCompound?` needs `masterPlate`; `discardExcess?` needs `basePlate.discardExcessPosition`; wash flags need the wash-configuration keys.
- **Assuming plate dictionaries share a shape**: `basePlate` has `startWells`/`endWells`/`wellSet`; `diluentPlate` and `masterPlate` instead have `columnOffset`/`rowOffset`/`reservoirSection` (and only `diluentPlate` has `dilutionRatio`) — copying keys across them is a common export drift.
