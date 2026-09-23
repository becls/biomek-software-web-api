# Fixed-8 Aspirate

| Property | Value |
|----------|-------|
| stepType | `"Fixed-8 Aspirate"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i3 (Fixed-8 pod required) |

## Behavior Summary

The Fixed-8 Aspirate step picks up liquid from a labware position using the Fixed-8 pod (8 channels in a column that move together). Tips must already be loaded before this step executes. The step accepts a manually specified technique via `prototype` (a stored technique name) or via `customPrototype` (an inline technique object). **Fixed-8 never auto-selects a technique — this is a permanent design choice.** Setting `autoSelectPrototype: true` raises an enqueue error. Always set `autoSelectPrototype: false` and name a technique. The labware must be a TiterPlate, TubeRack, or Reservoir with `CanPipette = true`.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | `expression-capable string` | — | Yes | — | Name of the Fixed-8 pod (e.g., `"Pod1"`). |
| `where` | `expression-capable string` | — | Yes | — | Deck position name or labware instance name where the source labware is located. |
| `amount` | `expression-capable string` | µL | Yes | — | Volume to aspirate. Must be non-negative and less than or equal to the volume in the well. |
| `firstWell` | `integer` | — | Conditional | — | 1-based well index to position the backmost tip over when aspirating. Used when `useWellExpression` is `false`. |
| `useWellExpression` | `boolean` | — | No | `false`| If `true`, use `firstWellExpression` instead of `firstWell`. |
| `firstWellExpression` | `expression-capable string` | — | Conditional | — | Expression for first well. Used when `useWellExpression` is `true`. |
| `liquidType` | `expression-capable string` | — | Yes | — | Liquid type for technique selection. Must not be `"Tip Contents"` for aspirate. Accepts `"Well Contents"` or any named liquid type. |
| `prototype` | `expression-capable string` | — | Conditional | — | Technique name. Required when `customPrototype` is not present. |
| `customPrototype` | `object` | — | Conditional | Not present | Inline technique object (a `_biomekType: "technique"` payload). When present, overrides `prototype`. |
| `autoSelectPrototype` | `boolean` | — | No | `false` | **Must be `false`** — Fixed-8 Aspirate does not support auto-selection by design (permanent). Setting `true` raises an enqueue error. |
| `overrideHeight` | `boolean` | — | No | `false` | If `true`, use custom `height` and `heightFrom` values. |
| `height` | `expression-capable string` | mm | Conditional | — | Custom pipetting height. Used when `overrideHeight` is `true`. |
| `heightFrom` | `expression-capable string` or integer | — | Conditional | — | Reference point for height. Accepts case-insensitive strings `"Liquid"`/`"Bottom"`/`"Top"` or integer forms `0`/`1`/`2` (same meaning). Used when `overrideHeight` is `true`. |
| `what` | `string` | — | No | `""` | Expected labware class name for the labware compatibility check (e.g., `"BCFlat96"`). When non-empty and the actual labware class at `where` differs, the compatibility check raises an enqueue error. Empty string skips the check. This field uses the short labware class name (e.g. `"BCFlat96"`); the `LabwareClasses\\` path prefix is used only in Instrument Setup's `deckItems`. |
| `emptyTips` | `boolean` | — | No | `false` | Not used for aspirate (dispense only). Should be `false`. |
| `mixCount` | `expression-capable string` | — | No | — | Ignored for Aspirate — Not applicable to this operation. |
| `podType` | `string` | — | No | `"Fixed8"` | Pod type identifier. Every editor-produced step carries `"Fixed8"`; do not change. |
| `operation` | `string` | — | **Yes** | — | **Must be present and equal `"Aspirate"`.** Every editor-produced step carries it, but when authoring from scratch, if the value doesn't match the step's operation (empty string when the key is omitted), enqueue raises an error. Include it when authoring from scratch. See rule 8. |
| `customHeight` | `boolean` | — | No | `false` | Encoding discriminator for `height`/`heightFrom`. When `false`, those keys carry numeric/enumerated values; when `true`, they carry expression text. Every editor-produced step carries it. Set it to match the form of `height` you author. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

> **Well numbering.** Wells are 1-based and **row-major** — on a 96-well plate A1 = 1, A12 = 12, B1 = 13, …, H12 = 96. The Fixed-8 pod places its lowest-numbered active probe at `firstWell` and strides **down the column by `+WellsX`**, so `firstWell` = 1 on a 96-well plate hits wells 1, 13, 25, 37, 49, 61, 73, 85 — all of column 1. To sweep plate columns, use `firstWellExpression` = `"=Col"`, **not** `"=1+(Col-1)*8"`: multiplying by the probe count assumes column-major numbering and overflows the plate. See [Probe & Mandrel Selection](../concept-guides/03-probe-and-mandrel-selection.md).

> **Liquid-relative heights need a locatable liquid surface.** A `heightFrom` of `"Liquid"` (or a follow-liquid technique) needs the run-time liquid level of the labware at `where`. Fixed-8 and Multichannel (96-/384-channel) pods do **not** have liquid-level sensing (LLS is Span-8 only), so an `"Unknown"` well cannot be sensed at run time on these pods — that labware's `volumeType` (seeded in Instrument Setup / Guided Setup) must be `"Nominal"` or `"Known"` **with an `evalAmounts` fill**, or the move throws at run time. See [Pipetting Techniques and Templates](../Pipetting-Techniques-and-Templates.md#liquid-types) §"Liquid types" / §"Liquid-relative heights need a locatable liquid surface".

## Enumerated / Constrained Values

### liquidType Values

| Value | Meaning |
|-------|---------|
| `"Well Contents"` | Auto-detect liquid from the labware's liquid data set at runtime |
| Any named liquid (e.g., `"Water"`, `"Serum"`) | Use the named liquid for technique selection |

> `"Tip Contents"` is NOT valid for aspirate — only for dispense.

### heightFrom Values

Accepted both as integers and case-insensitive strings:

| Integer | String alias | Meaning |
|---------|--------------|---------|
| `0` | `"Liquid"` | Height measured from the liquid surface |
| `1` | `"Bottom"` | Height measured from the well bottom |
| `2` | `"Top"` | Height measured from the well top |

## Cross-Field Validation Rules

1. `amount` **must** be non-negative — otherwise enqueue raises a negative-volume error.
2. When `useWellExpression` is `true`, `firstWellExpression` is used; otherwise `firstWell` is required.
3. `autoSelectPrototype` **must** be `false` — otherwise enqueue raises an auto-select-not-supported error.
4. Either `customPrototype` **must** be present or `prototype` **must** name a valid stored technique. If neither resolves to a technique, enqueue fails.
5. When `overrideHeight` is `true`, `height` and `heightFrom` must be provided.
6. `liquidType` **must not** be `"Tip Contents"` for aspirate.
7. The labware at `where` must be a TiterPlate, TubeRack, or Reservoir with `CanPipette = true`.
8. `operation` **must** be present and equal `"Aspirate"` — otherwise enqueue raises an operation-mismatch error (when the key is omitted, the value used is the empty string).
9. `amount` **must** keep the pod inside its pipetting range — a separate pod setting, 0–1050 µL in the factory configuration. The check runs at enqueue against the D position the move *ends* at, meaning the pod's current position plus the calibrated volume, so whatever the tips already hold counts against it. A breach raises `The D axis position of <n> µL is outside the supported pipetting range of the pod.` **No export carries that maximum**, and `settings.max.d` is not it: on a Fixed-8 the plunger and gripper share one motor, so `max.d` is the far end of the combined travel. Tip capacity is normally lower still and is what a method meets first, so this surfaces mainly with a computed volume or a large-capacity tip. See [Pod Settings Format](../biomek-file-formats/instrument-settings/pod-settings-format-spec.md#what-maxd-means-and-when-it-is-not-the-pipetting-ceiling) §"What `max.d` means, and when it is not the pipetting ceiling".

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

Basic aspirate from well A1 with a named technique:

```json
{
  "stepType": "Fixed-8 Aspirate",
  "parameters": {
    "pod": "Pod1",
    "where": "P3",
    "operation": "Aspirate",
    "amount": "100",
    "firstWell": 1,
    "liquidType": "Well Contents",
    "autoSelectPrototype": false,
    "prototype": "F8 Medium"
  }
}
```

Aspirate with explicit technique and height override:

```json
{
  "stepType": "Fixed-8 Aspirate",
  "parameters": {
    "pod": "Pod1",
    "where": "P5",
    "operation": "Aspirate",
    "amount": "50",
    "firstWell": 9,
    "liquidType": "Well Contents",
    "prototype": "F8 Medium",
    "autoSelectPrototype": false,
    "overrideHeight": true,
    "height": "2.0",
    "heightFrom": "Bottom"
  }
}
```

## Common Mistakes

- **Setting `autoSelectPrototype: true`** — Fixed-8 Aspirate/Dispense/Mix reject auto-selection at enqueue. Always pair `autoSelectPrototype: false` with an explicit `prototype` (or `customPrototype`).
- **Omitting `operation`** — every editor-produced step carries it, but when authoring from scratch, omission is read as the empty string and enqueue raises an error. Include `"operation": "Aspirate"`.
- **Using `"Tip Contents"` as `liquidType` for aspirate** — only `"Well Contents"` or a named liquid is valid here.
- **Omitting `prototype` with no `customPrototype`** — with auto-select disabled, one of them must resolve to a technique or enqueue fails.
- **Reading `settings.max.d` as the Fixed-8 pipetting ceiling** — it is the far end of the plunger/gripper travel the two share, not a pipetting limit. The pipetting maximum is a separate pod setting that no export carries; see rule 9.
