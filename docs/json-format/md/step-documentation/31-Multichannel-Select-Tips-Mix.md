# Multichannel Select Tips Mix

> **Reference manual:** Chapter 20 documents these as *Select Tips* steps (the manual omits the *Multichannel* prefix); the JSON `stepType` keeps the `Multichannel Select Tips …` name.

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Select Tips Mix"` |
| Category | Leaf (does not accept child steps) |
| Terminator | N/A |
| Compatible Hardware | Multichannel pod only |

## Behavior Summary

The Multichannel Select Tips Mix step performs repeated aspirate/dispense cycles at a single position to mix well contents. It uses the multichannel pod's currently loaded tip pattern. The mix count controls how many cycles are performed, and the volume controls how much liquid is moved per cycle.

This step must be placed inside a `"Multichannel Select Tips"` container, after tips have been loaded.

To perform a mix operation using a **full** head of tips (96 for a 96 head, 384
for a 384 head), do not use Multichannel Select Tips steps. Use Multichannel Mix.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `location` | `string` | — | Yes | `""` | Deck position name or labware **instance** name of source labware. Supports expressions (prefix with `=`). |
| `volume` | `expression-capable string` | µL | Yes | `"0"` | Volume per mix cycle. Must be non-negative and less than or equal to the volume in the labware. Unlike other mix steps, percentages are **not** accepted; volume must be in µL. |
| `mixCount` | `expression-capable string` | — | Yes | `"1"` | Number of mix cycles. Must be a positive integer. |
| `labwareClass` | `string` | — | Yes | `""` | Expected labware class at `location`. Empty or missing values raise a missing-labware-class error. A non-empty, valid class must be supplied. |
| `liquidtype` | `expression-capable string` | — | Yes | `""` | Source liquid classification. Commonly `"Well Contents"` (auto-detect from labware) or a named liquid type. Must be non-empty. |
| `columnOffset` | `expression-capable string` | — | Conditional | `"1"` | Starting column (1-based) to place the back left loaded tip over. Required when source labware is a microplate or tube rack. |
| `rowOffset` | `expression-capable string` | — | Conditional | `"1"` | Starting row (1-based) to place the back left loaded tip over. Required when source labware is a microplate or tube rack. |
| `reservoirSection` | `expression-capable string` | — | Conditional | `"1"` | Reservoir section number to place the back left loaded tip over. Required when source labware is a reservoir. |
| `prototype` | `string` | — | Conditional | `""` | Technique name. Required when `autoSelectPrototype` is `false` and `customPrototype` is not present. |
| `autoSelectPrototype` | `boolean` | — | No | `false` | If `true`, auto-select technique. |
| `customPrototype` | `object` | — | No | — | Inline technique object (`_biomekType: "technique"` payload). When present, overrides both `prototype` and `autoSelectPrototype`. Preserve on round-trip if present; prefer using named techniques to custom. |
| `overrideHeight` | `boolean` | — | No | `false` | If `true`, use custom height to aspirate. |
| `height` | `string` | mm | Conditional | — | Custom pipetting height. Required when `overrideHeight` is `true`. |
| `heightFrom` | integer or `string` | — | Conditional | — | Height reference — `0`/`1`/`2` or the aliases `"Liquid"`/`"Bottom"`/`"Top"`. Runtime accepts either form. Required when `overrideHeight` is `true`. |
| `doPositionOpenAndClose` | `boolean` | — | No | `true` | Whether to open the position before accessing and close when done. Preserve on round-trip if present; otherwise do not author. |
| `operation` | `string` | — | No | `"Mix"` | Defaults to `"Mix"`; do not change. |
| `customHeight` | `boolean` | — | No | `false` | Encoding discriminator for `height`: `false` = numeric offset; `true` = text/expression, which also implies `overrideHeight`. Use `false` with numeric heights and author `overrideHeight` / `height` / `heightFrom` directly. Preserve on round-trip if present. See [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#height-encoding-keys). |
| `usingPlate?` | `boolean` | — | No | — | Selects how the step renders in the printed/summary method — titer-plate (well column/row) vs. reservoir (section). **Keep it in sync with the actual labware type.** |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

## Cross-Field Validation Rules

1. This step **must** be inside a `"Multichannel Select Tips"` container.
2. Pod **must** have tips loaded, or a pod-no-tips error is raised.
3. Tips must have been loaded via Select Tips Load, else a wrong-tip-load error is raised.
4. `volume` must be non-negative, or a negative-volume error is raised.
5. Position must be reachable and contain labware.
6. Every active tip must map to a valid well, else a tip-miss error is raised. (Runtime motion collisions are a separate, later hardware check, not this enqueue validation.)
7. `mixCount` must evaluate to a positive integer *(throws error if empty or ≤ 0)*.
8. The per-cycle `volume` must not exceed the well's tracked contents — the same bound that governs any other aspirate, raising `Cannot pipette <v> uL; the well only has <x> uL in it.` `mixCount` does not multiply this budget, and no conditioning or pre-wet overhead is added to a mix cycle. See [Well Volume Tracking](../concept-guides/05-well-volume-tracking.md).
9. Tips must be empty before the mix begins, or the step raises `The mix
   operation cannot begin until tips are empty.` Air gaps do not trip the check;
   anything classified as liquid does.
10. When `overrideHeight` is `true`, `height` and `heightFrom` must be provided.
11. If `autoSelectPrototype` is `false`, then either `prototype` **must** name a valid technique or an inline `customPrototype` object **must** be bound. `customPrototype` takes precedence over both `autoSelectPrototype` and `prototype`.

## Structural Context

This step is a leaf inside a `"Multichannel Select Tips"` container. It does not
support subSteps.

## Canonical Examples

```json
{
  "stepType": "Multichannel Select Tips Mix",
  "parameters": {
    "location": "P4",
    "volume": "80",
    "mixCount": "5",
    "labwareClass": "BCFlat96",
    "columnOffset": "1",
    "rowOffset": "1",
    "liquidtype": "Well Contents",
    "autoSelectPrototype": true,
    "usingPlate?": true
  }
}
```

Mix in a reservoir section using expressions.
```json
{
  "stepType": "Multichannel Select Tips Mix",
  "parameters": {
    "location": "=reservoir",
    "volume": "=mixVolume",
    "mixCount": "=mixCount",
    "labwareClass": "ModularReservoir",
    "reservoirSection": "3",
    "liquidtype": "Well Contents",
    "autoSelectPrototype": true,
    "usingPlate?": false
  }
}
```

## Common Mistakes

- **`mixCount` of 0 or empty**: Must evaluate to a positive integer, else the step throws at enqueue.
- **Redundant technique keys**: `customPrototype` always wins when present (checked before `autoSelectPrototype`). When `customPrototype` is absent and `autoSelectPrototype: true`, `prototype` is ignored (auto-selection is used instead).
- **Tip-well mismatch**: The loaded tip pattern combined with `columnOffset`/`rowOffset` must land every active tip in a well (see rule 6).
- **Empty `labwareClass`**: The default is `""` (empty), which fails the non-empty check — supply a real class name.
- **Authoring `customHeight` or `operation`**: `customHeight` only selects how
  `height` is encoded — leave it `false` unless an expression is required, and carry the override on
  `overrideHeight` / `height` / `heightFrom`. `operation` defaults to
  `"Mix"` — leave it as-is.
