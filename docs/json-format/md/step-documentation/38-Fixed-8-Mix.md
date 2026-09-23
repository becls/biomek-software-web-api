# Fixed-8 Mix

| Property | Value |
|----------|-------|
| stepType | `"Fixed-8 Mix"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i3 (Fixed-8 pod required) |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).

## Behavior Summary

The Fixed-8 Mix step mixes liquid **in place** at a labware position using the Fixed-8 pod
(8 channels in a column that move together). It runs the selected technique's **Mix** operation `mixCount` times, aspirating and
dispensing `amount` µL at each mix cycle. Tips
must already be loaded before this step executes. Fixed-8 Mix shares its implementation
with Fixed-8 Aspirate/Dispense. **Like all Fixed-8 pipetting
steps, this step never auto-selects a technique.** Setting
`autoSelectPrototype: true` raises an enqueue error.
The labware must be a TiterPlate, TubeRack, or Reservoir with `CanPipette = true`.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | `expression-capable string` | — | Yes | — | Name of the Fixed-8 pod (e.g. `"Pod1"`). |
| `where` | `expression-capable string` | — | Yes | — | Deck position or labware instance name to mix at. |
| `amount` | `expression-capable string` | µL | Yes | — | Mix volume per cycle. Must be non-negative and less than or equal to the volume in the well. |
| `mixCount` | `expression-capable string` | — | Yes | `"1"` | **(Mix-only functional key)** Number of mix cycles. Evaluated to an integer. |
| `liquidType` | `expression-capable string` | — | Yes | — | Liquid type for technique selection. For Mix: `"Well Contents"` or a named liquid. **`"Tip Contents"` is NOT valid for Mix** (see below). |
| `firstWell` | `integer` | — | Conditional | — | 1-based well index where the first (backmost) loaded mandrel mixes. Used when `useWellExpression` is `false`. |
| `useWellExpression` | `boolean` | — | No | `false` | If `true`, use `firstWellExpression` instead of `firstWell`. |
| `firstWellExpression` | `expression-capable string` | — | Conditional | — | Well expression (integer or letter+number like `"B7"`). Used when `useWellExpression` is `true`. |
| `prototype` | `expression-capable string` | — | Conditional | — | Technique name. Required when `customPrototype` is not present. |
| `customPrototype` | `object` | — | Conditional | (absent) | Inline technique object (`_biomekType: "technique"`). When present, overrides `prototype`. |
| `autoSelectPrototype` | `boolean` | — | No | `false` | **Must be `false`** — Fixed-8 Mix does not support auto-selection. `true` raises an enqueue error. |
| `overrideHeight` | `boolean` | — | No | `false` | If `true`, use custom `height`/`heightFrom`; otherwise inherit from the technique's Mix settings. |
| `height` | `expression-capable string` | mm | Conditional | — | Custom pipetting height (used when `overrideHeight` is `true`). |
| `heightFrom` | `expression-capable string` or integer | — | Conditional | — | Height reference: case-insensitive `"Liquid"`/`"Bottom"`/`"Top"` or integer `0`/`1`/`2`. |
| `what` | `string` | — | No | `""` | Expected labware class name for the compatibility check. |
| `emptyTips` | `boolean` | — | No | `false` | **Inactive — not read at runtime for Mix.**. Preserve for round-trip; do not rely on it. |
| `podType` | `string` | — | No | `"Fixed8"` | Every editor-produced step carries `"Fixed8"`. Do not change. |
| `operation` | `string` | — | **Yes** | — | **Must be present and equal `"Mix"`.** Include it when authoring from scratch. |
| `customHeight` | `boolean` | — | No | `false` | Encoding discriminator for `height`/`heightFrom`. When `false`, those keys carry numeric/enumerated values; when `true`, they carry expression text. Every editor-produced step carries it. Set it to match the form of `height` you author. |

> Universal base keys (`caption`, `disabled`, …) — see
> [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md). Technique / height keys —
> see [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md).

> **Well numbering.** Wells are 1-based and **row-major** — on a 96-well plate A1 = 1, A12 = 12, B1 = 13, …, H12 = 96. The Fixed-8 pod places its lowest-numbered active probe at `firstWell` and strides **down the column by `+WellsX`**, so `firstWell` = 1 on a 96-well plate mixes wells 1, 13, 25, 37, 49, 61, 73, 85 — all of column 1. To sweep plate columns, use `firstWellExpression` = `"=Col"`, **not** `"=1+(Col-1)*8"`: multiplying by the probe count assumes column-major numbering and overflows the plate. See [Probe & Mandrel Selection](../concept-guides/03-probe-and-mandrel-selection.md).

> **Liquid-relative heights need a locatable liquid surface.** A `heightFrom` of `"Liquid"` (or a follow-liquid technique, including the technique's own Mix height) needs the run-time liquid level of the labware at `where`. Fixed-8 and Multichannel (96-/384-channel) pods do **not** have liquid-level sensing, so an `"Unknown"` well cannot be sensed at run time on these pods — that labware's `volumeType` (seeded in Instrument Setup / Guided Setup) must be `"Nominal"` or `"Known"` **with an `evalAmounts` fill**, or the mix throws at run time. See [Pipetting Techniques and Templates](../Pipetting-Techniques-and-Templates.md#liquid-types) §"Liquid types" / §"Liquid-relative heights need a locatable liquid surface".

## Enumerated / Constrained Values

### liquidType Values (Mix)

| Value | Meaning |
|-------|---------|
| `"Well Contents"` | Resolve the liquid type from the well(s) being mixed. |
| Any named liquid (e.g., `"Water"`, `"Serum"`) | Use the named liquid for technique selection. |

> **`"Tip Contents"` is NOT valid for Mix.** Using `"Tip Contents"` raises a dispense-only-liquid-type error. Mix's `liquidType` constraint matches Aspirate, not Dispense.

### heightFrom Values

Accepted as integers or case-insensitive strings:

| Integer | String alias | Meaning |
|---------|--------------|---------|
| `0` | `"Liquid"` | Height measured from the liquid surface |
| `1` | `"Bottom"` | Height measured from the well bottom |
| `2` | `"Top"` | Height measured from the well top |

## Cross-Field Validation Rules

1. **Operation must be `"Mix"`.** Enqueue compares the dictionary `operation` to
   the step's operation; mismatch raises an operation-mismatch error.
2. `amount` **must** be non-negative — otherwise enqueue raises a
   negative-volume error.
3. **`mixCount` is required and evaluated to an integer.**
4. **First well** — when `useWellExpression` is `true`, `firstWellExpression` is used and must
   resolve to a valid well;
   otherwise `firstWell` must be in `1..MaxWellNumber`.
5. **Auto-select is rejected.** `autoSelectPrototype: true` throws an auto-select-not-supported error. Provide `customPrototype` or a
   `prototype` naming a valid stored technique.
6. **Height override** — when `overrideHeight` is `true`, `height` and `heightFrom` are read;
   an unrecognized `heightFrom` raises an invalid-height-reference error.
7. **`liquidType` must not be `"Tip Contents"`** for Mix; it must resolve to an
   existing liquid type, else a no-liquid-type or ambiguous-liquid-type error is raised.
8. **Labware must support pipetting.** The labware at `where` must be a TiterPlate, TubeRack, or
   Reservoir with `CanPipette = true`.
9. **Tips must be loaded.** With no tips on the pod, an error is raised; mixed tip types raise a separate error.
10. The per-cycle `amount` must not exceed the well's tracked contents where the step runs, not the labware class's capacity — a mix cycle is an ordinary aspirate and takes the same below-empty check, raising `Cannot pipette <v> uL; the well only has <x> uL in it.` `mixCount` does not multiply this budget, and no conditioning or pre-wet overhead is added to a mix cycle. See [Well Volume Tracking](../concept-guides/05-well-volume-tracking.md).
11. Tips must be empty before the mix begins, or the step raises `The mix operation cannot begin until tips are empty.` Air gaps do not trip the check; anything classified as liquid does.

## Structural Context

Leaf step — no `subSteps`. Tips are not managed by this step; place it between the appropriate
Fixed‑8 Load Tips / Unload Tips steps.

## Canonical Example

Mix 50 µL, 3 cycles, at well A1 using Well Contents as the liquid type with a named technique:

```json
{
  "stepType": "Fixed-8 Mix",
  "parameters": {
    "pod": "Pod1",
    "where": "P3",
    "operation": "Mix",
    "amount": "50",
    "mixCount": "3",
    "firstWell": 1,
    "liquidType": "Well Contents",
    "autoSelectPrototype": false,
    "prototype": "F8 Medium",
    "overrideHeight": false
  }
}
```

Editor-faithful form (includes the inert/non-functional keys the editor writes on every save):

```json
{
  "stepType": "Fixed-8 Mix",
  "parameters": {
    "podType": "Fixed8",
    "operation": "Mix",
    "dynamic?": true,
    "pod": "Pod1",
    "where": "P5",
    "what": "BCFlat96",
    "amount": "50",
    "mixCount": "5",
    "liquidType": "Well Contents",
    "useWellExpression": false,
    "firstWell": 9,
    "firstWellExpression": "",
    "autoSelectPrototype": false,
    "prototype": "F8 Medium",
    "overrideHeight": true,
    "height": 2.0,
    "heightFrom": 1,
    "customHeight": false,
    "emptyTips": false
  }
}
```

## Common Mistakes

- **Setting `autoSelectPrototype: true`** — rejected at enqueue for all Fixed-8 steps. Pair `autoSelectPrototype: false` with an explicit `prototype` (or `customPrototype`).
- **Using `"Tip Contents"` as `liquidType`** — Mix matches Aspirate here, not Dispense; enqueue raises an error. Use `"Well Contents"` or a named liquid.
- **Expecting `emptyTips` to do anything for Mix** — it is Dispense-only; the editor writes `false` on every Mix step and execution ignores it.
- **Omitting `mixCount` when authoring from scratch** — every editor-produced step carries it (default `1`), but a hand-built dictionary without the key raises a mix-count-not-specified error at enqueue.
- **Reading `settings.max.d` as the Fixed-8 pipetting ceiling** — it is the far end of the plunger/gripper travel the two share, not a pipetting limit. The pipetting maximum is a separate pod setting that no export carries; see rule 10.

## Porting: There Is No Span-8 Mix Step

When converting a Fixed-8 method to Span-8, there is **no `"Span-8 Mix"` step type**.

**Preferred path — the technique.** Mixing in place on Span-8 is configured on the pipetting
technique, not with a step key.
Select a technique that has is configured to mix before aspirate or mix after dispense with `prototype` on the Span-8 step that does the pipetting. See
**Mixing in place** in [Transfer](11-Transfer.md#mixing-in-place). An item-level
`mixCount` on a `Transfer` item does **not** produce a mix — it is inert on every pod family.

**Explicit-loop path.** If you instead expand the mix into `Span-8 Aspirate` + `Span-8 Dispense`
inside a `Loop` around the same well, note that neither Span-8 Aspirate nor Span-8 Dispense
reads a `mixCount` key — the mix count of a Fixed-8 Mix step has no counterpart on those two
steps and simply doesn't apply. When porting, therefore:

- drop `mixCount` **and** the `operation:"Mix"` value from the Fixed-8 step — the loop's
  iteration count replaces `mixCount` in this expansion, and `operation` on the Span-8
  Aspirate/Dispense steps is `"Aspirate"`/`"Dispense"`, not `"Mix"`.
- set `liquidType:"Well Contents"` on the aspirate and `"Tip Contents"` on the dispense;
- **change every `prototype` from `F8 …` to an `S8 …` technique** (see the F8/S8/MC
  technique-family rule in `Pipetting-Techniques-and-Templates.md` — a Fixed-8 technique
  on a Span-8 step will render the sub-steps incorrectly in the editor even when the JSON is
  otherwise valid).

Multichannel has its own `Multichannel Mix` step (`24-Multichannel-Mix.md`); this porting note
applies to Span-8 conversions only.
