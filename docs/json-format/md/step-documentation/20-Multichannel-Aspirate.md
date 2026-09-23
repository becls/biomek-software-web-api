# Multichannel Aspirate

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Aspirate"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5 or i7 with a Multichannel pod |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Universal base keys (`caption`, `dynamic?`, `disabled`, …) are **not** re-documented here.

---

## Behavior Summary

The Multichannel Aspirate step removes liquid from source labware using a fixed
multichannel head (96-channel or 384-channel). At enqueue time the step
validates that the specified pod is a multichannel pod (only multichannel pods
are accepted; a Span-8 or Fixed-8 pod will produce an error. It resolves the deck position, verifies that labware is present and on top
of any stack at that position, checks that the labware class is compatible with
the actual labware at the position, and confirms the labware class has a "Can
Pipette" characteristic. If `refreshTips` is `true`, existing tips are discarded
and fresh tips are loaded from the specified tip labware class. It then uses the
given pipetting technique to aspirate the given volume from the selected wells.

After aspirating with a Multichannel Aspirate step, use a Multichannel Dispense
step to dispense the liquid.

Note that Multichannel Aspirate, Multichannel Dispense, Multichannel Mix, and
Multichannel Wash Tips share much of their implementation. As a result, some
parameters are listed here that are only used by those other steps. See each
step's documentation for details.

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `operation` | string | — | Yes | `"Aspirate"` | Step action identity. Must be `"Aspirate"`. |
| `pod` | string | — | Yes | `""` | Name of the multichannel pod to use (e.g., `"Pod1"`). Must resolve to a multichannel pod. The MC pod is `"Pod1"` on a standalone Multichannel instrument (i5-MC, i7-MC) and on an i7 Hybrid; it is `"Pod2"` only when the MC is the right-hand pod of a dual-pod i7. Never author `"MC"`/`"MPod"`. See [Introduction to Biomek Method JSON](../Introduction-to-Biomek-Method-JSON.md#instrument-variants) §"Instrument Variants" / "Pod Types". |
| `location` | string | — | Yes | `""` | Deck position or labware name where the source labware is located. Supports expressions (prefix with `=`). Accepts a deck-position name (e.g. `P5`) or a labware **instance** name (assigned via the labware's `properties.name` in Instrument Setup) — matched case-insensitively. Example 1's `"SourcePlate"` refers to a placed labware instance named "SourcePlate". See [Instrument Setup](03-Instrument-Setup.md) and [Labware and Position Resolution](../concept-guides/02-labware-and-position-resolution.md). |
| `labwareClass` | string | — | Yes | `""` | Expected labware class at the source position (e.g., `"BCFlat96"`). Must be compatible with the actual labware. This field uses the short labware class name; the `LabwareClasses\\` path prefix is used only in Instrument Setup's `deckItems`. |
| `liquidtype` | string | — | Yes | `""` | Source liquid classification. Commonly `"Well Contents"` (auto-detect from labware) or a named liquid type. Must be non-empty. |
| `volume` | number or string | µL | Yes | `0.0` | Amount to aspirate. Positive numeric value or expression string (e.g., `"20"`, `"=AspVol"`). **An aspirate cannot drive the plunger past the installed head's capacity** — the pod's D-axis maximum, in µL (e.g. 300 for the 300 µL MC-96 head). The limit applies to the plunger position the move *ends* at — the pod's current D position plus this move's calibrated volume — so whatever the tips already hold counts against it: a 200 µL aspirate on top of 150 µL already in the tips overruns a 300 µL head even though neither volume alone does. Violations fail at enqueue, before any motion, with `Move of <pod> to D of <n> µL violates D axis maximum of <max> µL.` Read the ceiling from `podSettings.<pod>.settings.max.d` in the instrument-settings export — see [Pod Settings Format](../biomek-file-formats/instrument-settings/pod-settings-format-spec.md#what-maxd-means-and-when-it-is-not-the-pipetting-ceiling) §"What `max.d` means, and when it is not the pipetting ceiling". Watch computed volumes (base + conditioning/excess terms) that quietly push one aspirate over the ceiling. Separately, an aspirate also cannot exceed the loaded **tip's** own capacity (e.g. 230 µL for a BC230 tip) — that is a different limit and fails with `Too much aspirated into tip(s); max volume is <n> µL and a total of <m> µL has been aspirated.` |
| `selectionInfo` | comArray (`arraySubtype: "integer"`) | — | Conditional | `""` (empty) | Array of 1-based section/quadrant indices to aspirate from. On these low-level Multichannel steps `selectionInfo` is a `comArray` of `arraySubtype: "integer"` carrying section/quadrant indices (e.g. `[1]`) — **different** from the item-level `selectionInfo` on Transfer/Combine, which is a well selection whose subtype varies by pod (boolean well-mask on Fixed-8, integer on Span-8/MC). **Must contain exactly one element** — the step requires exactly one target group per invocation (see Cross-Field rule 11); to process multiple sections, use a Loop with `sectionExpression`. Required when `useExpression` is `false`. Serialized as `{"_biomekType": "comArray", "arraySubtype": "integer", "values": [...]}`. |
| `useExpression` | boolean | — | No | `false` | When `true`, selection is determined by `sectionExpression` instead of `selectionInfo`. |
| `sectionExpression` | string | — | Conditional | `""` | Expression used for section/column/quadrant selection at runtime. Required when `useExpression` is `true`. |
| `refreshTips` | boolean | — | Yes | `false` | Whether to discard current tips and load fresh tips before aspirating. |
| `tipLabwareClass` | string | — | Conditional | `""` | The tips to use, specified either as a labware class, a labware instance name, or a position name. Expression capable. Required when `refreshTips` is `true`. |
| `autoSelectPrototype` | boolean | — | Yes | `false` | Whether the software auto-selects a technique based on pod type, tip type, labware, liquid type, and volume. |
| `prototype` | string | — | Conditional | `""` | Explicit technique name. Required when `autoSelectPrototype` is `false` and `customPrototype` is not present. |
| `customPrototype` | object | — | No | Not present | Inline technique object (`_biomekType: "technique"` payload). When present, overrides both `prototype` and `autoSelectPrototype`. |
| `overrideHeight` | boolean | — | Yes | `false` | Whether to override the technique's default aspirate height. |
| `height` | number or string | mm | Conditional | `0.0` | Height offset from the reference point. Required when `overrideHeight` is `true`. Supports expressions. |
| `heightFrom` | integer or string | — | Conditional | `0` | Reference point for height: `0` = liquid surface, `1` = well bottom, `2` = well top. Also accepts string aliases `"Liquid"`, `"Bottom"`, `"Top"`. |

> **A liquid-relative height needs a locatable liquid surface.** With `heightFrom` = `0`/`"Liquid"` (or a technique that follows the liquid), the target labware's surface must be computable at run time: its `volumeType` — set in Instrument Setup / Guided Setup — must be `"Nominal"` or `"Known"` **with** an `evalAmounts` fill. Multichannel (96-/384-channel) and Fixed-8 pods do **not** support liquid-level sensing (LLS is Span-8 only), so on those pods an `"Unknown"` well cannot be rescued at run time — the move throws. See [Pipetting Techniques and Templates](../Pipetting-Techniques-and-Templates.md#liquid-types) §"Liquid types" / §"Liquid-relative heights need a locatable liquid surface".

### Editor-persisted keys (not read by the Aspirate engine)

These are written on every export by the editor but are **not** read. Include them to match editor output; classify as **inert UI-state**.

| Key | Type | Units | Default | Description / classification |
|-----|------|-------|---------|------------------------------|
| `customHeight` | boolean | — | `false` | Encoding discriminator for `height`: `false` = numeric offset; `true` = text/custom height, which also implies `overrideHeight`. Use `false` with numeric heights. See [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#height-encoding-keys). |
| `pattern` | comArray (`arraySubtype: "boolean"`) | — | (well-count mask) | **Inert UI-state.** Boolean well-selection mask maintained by the labware view to restore the visual selection. Engine uses `selectionInfo`, not `pattern`. |
| `aspirateTime` | expression-capable string | s | `"1"` | **Inactive and obsolete.** This key is not used. |
| `mixCount` | expression-capable string | — | `"1"` | **Inert UI-state on Aspirate.** This key is only used for the Multichannel Mix step. Not read by the Aspirate engine. |
| `washSettleTime` | expression-capable string | s | `"0"` | **Inactive and obsolete.** This key is not used. |

---

## Enumerated / Constrained Values

| Parameter | Allowed Values | Notes |
|-----------|---------------|-------|
| `operation` | `"Aspirate"` | Must always be `"Aspirate"` for this step type. |
| `heightFrom` | `0`, `1`, `2` (or `"Liquid"`, `"Bottom"`, `"Top"`) | Integer form preferred in JSON. String aliases are resolved at runtime. |
| `liquidtype` | Any named liquid, `"Well Contents"` | `"Tip Contents"` is **invalid** for aspirate operations. |
| `selectionInfo` values | Positive integers (1-based) | The array iterates the *sections* the head touches, so the valid values match how many sections the labware exposes to the head — **not** the plate's column count. For a **96-well plate + 96-channel head** there is exactly one section, so only `[1]` is valid; authoring `[1,2,3]` fails because the extra indices do not match sections the head exposes on that labware. For a **384-well plate + 96-channel head** or **1536-well plate + 384-channel head** each labware exposes four interleaved quadrants and the valid values are quadrant indices `1`–`4`. However, the Multichannel Aspirate step requires that there be only a single section used, so this array must have exactly one value. |

---

## Cross-Field Validation Rules

1. `operation` **must** equal `"Aspirate"`. The step defaults to this value.
2. `pod` **must** resolve to a Multichannel pod. A Span-8 pod or Fixed-8 pod will produce a runtime error.
3. `location` **must** resolve to a valid, reachable deck position with labware present and on top of any stack.
4. `labwareClass` **must** be compatible with the actual labware class at the resolved position.
5. The labware class **must** have a `"Can Pipette"` characteristic, or the step raises a runtime error
6. If `refreshTips` is `true`, then `tipLabwareClass` **must** be a non-empty
   string naming a valid tip labware class, tip box instance name, or position name.
7. If `autoSelectPrototype` is `false`, then `prototype` **must** be a non-empty string naming a valid technique. If the technique cannot be found, the step raises a technique-not-found error.
8. If `autoSelectPrototype` is `true`, at least one matching technique must exist for the given context (pod type, head type, tip class, labware, liquid type, volume), or the step raises a technique-selection error.
9. If `overrideHeight` is `true`, both `height` and `heightFrom` **must** be provided with valid values.
10. `heightFrom` should be `0`, `1`, or `2` (or `"Liquid"`, `"Bottom"`, `"Top"`). Unrecognized string values raise an invalid-height-from error. Any `heightFrom` value outside `{0, 1, 2}` raises the same height-from error at run time when `overrideHeight` is `true`.
11. `selectionInfo` (or the result of `sectionExpression`) **must** resolve to
    exactly one valid target group (either 1 for a whole plate where the plate
    wells match the head geometry, or a quadrant if the plate has more wells
    than the head). Zero targets or multiple targets raise a
    selection error.
12. If `useExpression` is `true`, then `sectionExpression` **must** be a non-empty, evaluable expression string. If `false`, `selectionInfo` must be a non-empty array of valid section indices.
13. `liquidtype` **must not** be `"Tip Contents"` for an aspirate step. Use `"Well Contents"` or a named liquid type.
14. When `liquidtype` is `"Well Contents"`, all wells in the target group must contain the same liquid type at runtime, or the step raises a liquid-mismatch error.
15. `labwareClass` must be reachable/pipettable by the pod. An incompatible labware+pod combination raises a labware-access error.

---

## Structural Context

This step is a leaf and does not support subSteps.

---

## Canonical Examples

### Example 1: Basic Multichannel Aspirate from all wells of a 96-well plate (auto-select technique)

Includes the editor-persisted keys so it matches a real export.

```json
{
  "stepType": "Multichannel Aspirate",
  "parameters": {
    "operation": "Aspirate",
    "pod": "Pod1",
    "location": "SourcePlate",
    "labwareClass": "BCFlat96",
    "liquidtype": "Well Contents",
    "volume": 20,
    "selectionInfo": {
      "_biomekType": "comArray",
      "arraySubtype": "integer",
      "values": [1]
    },
    "useExpression": false,
    "sectionExpression": "",
    "refreshTips": false,
    "tipLabwareClass": "",
    "autoSelectPrototype": true,
    "prototype": "",
    "overrideHeight": false,
    "height": 0.0,
    "heightFrom": 0,
    "customHeight": false,
    "pattern": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": []
    },
    "aspirateTime": "1",
    "mixCount": "1",
    "washSettleTime": "0"
  }
}
```

> The `pattern` array is shown empty for brevity; in a real export it is a full-length boolean mask matching the labware's well count. The engine ignores `pattern` and uses `selectionInfo`.

### Example 2: Aspirate from quadrant 2 of a 384-well plate with tip refresh, explicit technique, and height override

```json
{
  "stepType": "Multichannel Aspirate",
  "parameters": {
    "operation": "Aspirate",
    "pod": "Pod1",
    "location": "P4",
    "labwareClass": "CostarFlat384Square",
    "liquidtype": "Serum",
    "volume": "=AspVol",
    "selectionInfo": {
      "_biomekType": "comArray",
      "arraySubtype": "integer",
      "values": [2]
    },
    "useExpression": false,
    "sectionExpression": "",
    "refreshTips": true,
    "tipLabwareClass": "BC230",
    "autoSelectPrototype": false,
    "prototype": "MC",
    "overrideHeight": true,
    "height": -2.0,
    "heightFrom": 0
  }
}
```

### Example 3: Aspirate from a 384-well plate using expression-based quadrant selection

```json
{
  "stepType": "Multichannel Aspirate",
  "parameters": {
    "operation": "Aspirate",
    "pod": "Pod1",
    "location": "P7",
    "labwareClass": "CostarFlat384Square",
    "liquidtype": "Water",
    "volume": 5,
    "selectionInfo": {
      "_biomekType": "comArray",
      "arraySubtype": "integer",
      "values": []
    },
    "useExpression": true,
    "sectionExpression": "=CurrentQuadrant",
    "refreshTips": false,
    "tipLabwareClass": "",
    "autoSelectPrototype": true,
    "prototype": "",
    "overrideHeight": true,
    "height": 1.5,
    "heightFrom": 1
  }
}
```

---

## Common Mistakes

- **Confusing column indices with well indices**: For a 96-well plate with a
  96-channel head, `selectionInfo` values are section indices (1-only on 96-well
  with 96-format head or 384-well with 384-format head; quadrants 1–4 on
  384-well with 96-format head or 1536-well with 384-format head), not individual well numbers. For a 384-well
  plate with a 96-channel head, they are **quadrant indices** (1–4).
- **Selection that resolves to more than one target group**: The selection resolver must produce exactly one target — non-contiguous selections or incompatible section indices raise a selection error.
- **Wrong sign convention for `height`**: From Liquid or Top the offset is negative (below reference); from Bottom it is positive.
- **Confusing `useExpression` with Span-8 well-expression keys**: Here it toggles `sectionExpression` vs `selectionInfo`. There is no `useWellExpression`/`firstWellExpression` on this step.
- **Assuming `autoSelectPrototype` defaults to `true`**: It defaults to `false`. Set it explicitly or `prototype` is required.

---

## Related Guides

- **[Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md)** — Technique auto-selection algorithm, height override mechanics, `C__` pipetting variable reference, and template architecture. Covers how `autoSelectPrototype`, `prototype`, `overrideHeight`, `height`, and `heightFrom` interact with the technique system.
