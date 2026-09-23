# Multichannel Dispense

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Dispense"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5 or i7 with a Multichannel pod |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Universal base keys (`caption`, `dynamic?`, `disabled`, …) are **not** re-documented here.

---

## Behavior Summary

The Multichannel Dispense step dispenses liquid into destination labware using a
fixed multichannel head (96-channel or 384-channel). At enqueue time the step
validates that the specified pod is a multichannel pod (only multichannel pods
are accepted; a Span-8 or Fixed-8 pod will produce an error). It resolves the deck position, verifies that
labware is present and on top of any stack at that position, checks that the
labware class is compatible with the actual labware at that position,
confirms the labware class has a "Can Pipette" characteristic, and verifies that
tips are loaded. It then uses the given pipetting technique to dispense the
given volume, or the volume in tips if `emptyTips` is `true`, into the selected
wells.

Use the Multichannel Dispense step to dispense liquid that was aspirated with a
Multichannel Aspirate step.

Note that Multichannel Aspirate, Multichannel Dispense, Multichannel Mix, and
Multichannel Wash Tips share much of their implementation. As a result, some
parameters are listed here that are only used by those other steps. See each
step's documentation for details.

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `operation` | string | — | Yes | `"Dispense"` | Step action identity. Must be `"Dispense"`. |
| `pod` | string | — | Yes | `""` | Name of the multichannel pod to use (e.g., `"Pod1"`). Must resolve to a multichannel pod. The MC pod is `"Pod1"` on a standalone Multichannel instrument (i5-MC, i7-MC) and on an i7 Hybrid; it is `"Pod2"` only when the MC is the right-hand pod of a dual-pod i7. Never author `"MC"`/`"MPod"`. See [Introduction to Biomek Method JSON](../Introduction-to-Biomek-Method-JSON.md#instrument-variants) §"Instrument Variants" / "Pod Types". |
| `location` | string | — | Yes | `""` | Deck position or labware name where the destination labware is located. Supports expressions (prefix with `=`). Accepts a deck-position name (e.g. `P5`) or a labware **instance** name (assigned via the labware's `properties.name` in Instrument Setup) — matched case-insensitively. Example 1's `"DestPlate"` refers to a placed labware instance named "DestPlate". See [Instrument Setup](03-Instrument-Setup.md) and [Labware and Position Resolution](../concept-guides/02-labware-and-position-resolution.md). |
| `labwareClass` | string | — | Yes | `""` | Expected labware class at the destination position (e.g., `"BCFlat96"`). Must be compatible with the actual labware. This field uses the short labware class name; the `LabwareClasses\\` path prefix is used only in Instrument Setup's `deckItems`. |
| `liquidtype` | string | — | Yes | `""` | Destination liquid classification. Commonly `"Tip Contents"` (auto-detect from tips) or a named liquid type. Must be non-empty. |
| `volume` | number or string | µL | Yes (key must be present) | `"-1"` | Amount to dispense. Positive numeric value or expression string. The **key must be present**, even when `emptyTips` is `true`, in which case the authored value is otherwise ignored for the dispense amount. Default `"-1"` is a sentinel; authored values must be positive. **A dispense is bounded on the D-axis *minimum*, not the maximum**: it drives the plunger down, away from the head's capacity, so the head rating in `podSettings.<pod>.settings.max.d` is not the limit here — that ceiling applies to aspirates. Driving the plunger below the pod's D-axis minimum fails at enqueue with `Move of <pod> to D of <n> µL violates D axis minimum of <min> µL.`, though in practice rule 16 (dispensing more than the tips hold) catches the same authoring error first. See [Pod Settings Format](../biomek-file-formats/instrument-settings/pod-settings-format-spec.md#what-maxd-means-and-when-it-is-not-the-pipetting-ceiling) §"What `max.d` means, and when it is not the pipetting ceiling". |
| `emptyTips` | boolean | — | Yes (key must be present) | `false` | When `true`, the step ignores `volume` and dispenses the entire tip contents (trailing air gap + liquid + blow-out) at the initial dispense height, overriding technique "Follow liquid level" settings. The **key must be present**; always author it. If technique height and following liquid are important, set `emptyTips` to `false` and ensure that `volume` is set to the actual volume in the tips. |
| `selectionInfo` | comArray (`arraySubtype: "integer"`, long[]) | — | Conditional | `""` (empty) | Array of 1-based section or quadrant indices to dispense into. On these low-level Multichannel steps `selectionInfo` is a `comArray` of `arraySubtype: "integer"` carrying section/quadrant indices (e.g. `[1]`) — **different** from the item-level `selectionInfo` on Transfer/Combine, which is a well selection whose subtype varies by pod (boolean well-mask on Fixed-8, integer on Span-8/MC). Required when `useExpression` is `false`. Serialized as `{"_biomekType": "comArray", "arraySubtype": "integer", "values": [...]}`. **Must contain exactly one element** — the step requires exactly one target group per invocation; to process multiple sections, use a Loop with `sectionExpression` or use multiple Multichannel Dispense steps. |
| `useExpression` | boolean | — | No | `false` | When `true`, selection is determined by `sectionExpression` instead of `selectionInfo`. |
| `sectionExpression` | string | — | Conditional | `""` | Expression used for section/column/quadrant selection at runtime. Required when `useExpression` is `true`. |
| `refreshTips` | boolean | — | Yes | `false` | **Inactive.** This key must be authored, but it is ignored by the dispense step. Always author as `false`. |
| `tipLabwareClass` | string | — | No | `""` | **Inactive.** This key is used only in aspirate, not in dispense steps. Do not author it. |
| `autoSelectPrototype` | boolean | — | Yes | `false` | Whether the software auto-selects a technique based on pod type, head type, tip class, labware, liquid type, and volume. |
| `prototype` | string | — | Conditional | `""` | Explicit technique name. Required when `autoSelectPrototype` is `false` and `customPrototype` is not present. |
| `customPrototype` | object (`_biomekType: "technique"`) | — | No | Not present | Inline technique object; when present and `autoSelectPrototype` is `false`, used instead of the named `prototype`. When `autoSelectPrototype` is `true`, `customPrototype` is ignored. |
| `overrideHeight` | boolean | — | Yes | `false` | Whether to override the technique's default dispense height. |
| `height` | number or string | mm | Conditional | `0.0` | Height offset from the reference point. Required when `overrideHeight` is `true`. Supports expressions. |
| `heightFrom` | integer or string | — | Conditional | `0` | Reference point for height: `0` = liquid surface, `1` = well bottom, `2` = well top. Also accepts string aliases `"Liquid"`, `"Bottom"`, `"Top"`. |

> **A liquid-relative height needs a locatable liquid surface.** With `heightFrom` = `0`/`"Liquid"` (or a technique that follows the liquid), the target labware's surface must be computable at run time: its `volumeType` — set in Instrument Setup / Guided Setup — must be `"Nominal"` or `"Known"` **with** an `evalAmounts` fill. Multichannel (96-/384-channel) and Fixed-8 pods do **not** support liquid-level sensing (LLS is Span-8 only), so on those pods an `"Unknown"` well cannot be rescued at run time — the move throws. See [Pipetting Techniques and Templates](../Pipetting-Techniques-and-Templates.md#liquid-types) §"Liquid types" / §"Liquid-relative heights need a locatable liquid surface".

### Editor-persisted keys (not read by the Dispense engine)

These are written on every export by the editor but are **not** read at enqueue. Include them to match editor output; classify as **inert UI-state**.

| Key | Type | Units | Default | Description / classification |
|-----|------|-------|---------|------------------------------|
| `customHeight` | boolean | — | `false` | Encoding discriminator for `height`: `false` = numeric offset; `true` = text/custom height, which also implies `overrideHeight`. Use `false` with numeric heights. See [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#height-encoding-keys). |
| `pattern` | comArray (`arraySubtype: "boolean"`) | — | (well-count mask) | **Inert UI-state.** Boolean well-selection mask maintained by the labware view to restore the visual selection. Engine uses `selectionInfo`, not `pattern`. |
| `aspirateTime` | expression-capable string | s | `"1"` | **Inactive and obsolete.** This key is not used. |
| `mixCount` | expression-capable string | — | `"1"` | **Inert UI-state on Dispense.** This key is only used for the Multichannel Mix step. Not read by the Dispense engine. |
| `washSettleTime` | expression-capable string | s | `"0"` | **Inactive and obsolete.** This key is not used. |

---

## Enumerated / Constrained Values

| Parameter | Allowed Values | Notes |
|-----------|---------------|-------|
| `operation` | `"Dispense"` | Must always be `"Dispense"` for this step type. |
| `heightFrom` | `0`, `1`, `2` (or `"Liquid"`, `"Bottom"`, `"Top"`) | Integer form preferred in JSON. String aliases are resolved at runtime. |
| `liquidtype` | Any named liquid, `"Tip Contents"` | `"Tip Contents"` is the editor default for dispense. `"Well Contents"` is **invalid** for dispense operations. |
| `selectionInfo` values | Positive integers (1-based) | The array iterates the *sections* the head touches, so the valid values match how many sections the labware exposes to the head — **not** the plate's column count. For a **96-well plate + 96-channel head** there is exactly one section, so only `[1]` is valid; authoring `[1,2,3]` fails because the extra indices do not match sections the head exposes on that labware. For a **384-well plate + 96-channel head** or **1536-well plate + 384-channel head** each labware exposes four interleaved quadrants and the valid values are quadrant indices `1`–`4`. However, the Multichannel Dispense step requires that there be only a single section used, so this array must have exactly one value. |

---

## Cross-Field Validation Rules

1. `operation` **must** equal `"Dispense"`. The step defaults to this value.
2. `pod` **must** resolve to a Multichannel pod. A Span-8 or Fixed-8 pod will produce a runtime error.
3. `location` **must** resolve to a valid, reachable deck position with labware present and on top of any stack.
4. `labwareClass` **must** be compatible with the actual labware class at the resolved position.
5. The labware class **must** have a `"Can Pipette"` characteristic, or the step raises a runtime error.
6. Tips **must** be loaded on the pod at runtime. If no tips are present, the step raises a no-tips error.
7. If `emptyTips` is `false`, `volume` **must** be a positive value. When `emptyTips` is `true`, `volume` is ignored and may be any value.
8. If `autoSelectPrototype` is `false`, then `prototype` **must** be a non-empty string naming a valid technique. If the technique cannot be found, the step raises a technique-not-found error.
9. If `autoSelectPrototype` is `true`, at least one matching technique must exist for the given context (pod type, head type, tip class, labware, liquid type, volume), or the step raises a technique-selection error.
10. If `overrideHeight` is `true`, both `height` and `heightFrom` **must** be provided with valid values.
11. `heightFrom` should be `0`, `1`, or `2` (or `"Liquid"`, `"Bottom"`, `"Top"`). Unrecognized string values raise an invalid-height-from error. Any `heightFrom` value outside `{0, 1, 2}` raises the same height-from error at run time when `overrideHeight` is `true`.
12. `selectionInfo` (or the result of `sectionExpression`) **must** resolve to
    exactly one valid target group (either 1 for a whole plate where the plate
    wells match the head geometry, or a quadrant if the plate has more wells
    than the head). Zero targets or multiple targets raise a
    selection error.
13. If `useExpression` is `true`, then `sectionExpression` **must** be a non-empty, evaluable expression string. If `false`, `selectionInfo` must be a non-empty array of valid section indices.
14. `liquidtype` **must not** be `"Well Contents"` for a dispense step. Use `"Tip Contents"` or a named liquid type.
15. When `liquidtype` is `"Tip Contents"` and the tips contain no non-air contents, the liquid type falls back to the labware's liquid dataset for the first well in the target group.
16. `volume` **must not** exceed the current tip contents: dispensing more than the tips hold raises a dispense-too-much error.
17. The dispensed volume **must not** overflow the destination well: exceeding well capacity raises a well-overflow error.
18. `labwareClass` must be reachable/pipettable by the pod, else the step raises a labware-access error.

---

## Structural Context

This step is a leaf and does not support subSteps.

---

## Canonical Examples

### Example 1: Basic Multichannel Dispense to section 1 of a 96-well plate (auto-select technique)

Includes the editor-persisted keys so it matches a real export.

```json
{
  "stepType": "Multichannel Dispense",
  "parameters": {
    "operation": "Dispense",
    "pod": "Pod1",
    "location": "DestPlate",
    "labwareClass": "BCFlat96",
    "liquidtype": "Tip Contents",
    "volume": 20,
    "emptyTips": false,
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

### Example 2: Dispense with Empty Tips, height override, and explicit technique

```json
{
  "stepType": "Multichannel Dispense",
  "parameters": {
    "operation": "Dispense",
    "pod": "Pod1",
    "location": "P6",
    "labwareClass": "BCDeep96Square",
    "liquidtype": "Tip Contents",
    "volume": 0,
    "emptyTips": true,
    "selectionInfo": {
      "_biomekType": "comArray",
      "arraySubtype": "integer",
      "values": [1]
    },
    "useExpression": false,
    "sectionExpression": "",
    "refreshTips": false,
    "tipLabwareClass": "",
    "autoSelectPrototype": false,
    "prototype": "MC",
    "overrideHeight": true,
    "height": -1.0,
    "heightFrom": 2
  }
}
```

### Example 3: Expression-based selection dispensing to a quadrant of a 384-well plate defined in a variable

```json
{
  "stepType": "Multichannel Dispense",
  "parameters": {
    "operation": "Dispense",
    "pod": "Pod1",
    "location": "P7",
    "labwareClass": "CostarFlat384Square",
    "liquidtype": "Water",
    "volume": "=DispenseVol",
    "emptyTips": false,
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
    "overrideHeight": false,
    "height": 0.0,
    "heightFrom": 0
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
- **Empty Tips height behavior is not "follow liquid"**: With `emptyTips: true` the technique's Follow-Liquid-Level setting is overridden — the entire tip contents (trailing air gap + liquid + blow-out) are expelled at the *initial* dispense height. The tip does not track the surface downward.
- **Expecting `volume` to matter when `emptyTips` is `true`**: The runtime sums all tip content volumes (including air gaps) and uses that for technique selection; the authored `volume` is bypassed entirely. Set it to `0` for clarity.
- **Confusing `useExpression` with Span-8 well-expression keys**: Here it toggles `sectionExpression` vs `selectionInfo`. There is no `useWellExpression`/`firstWellExpression` on this step.
- **Assuming `autoSelectPrototype` defaults to `true`**: It defaults to `false`. Set it explicitly or `prototype` is required.

---

## Related Guides

- **[Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md)** — Technique auto-selection algorithm, height override mechanics, `C__` pipetting variable reference, and template architecture. Covers how `autoSelectPrototype`, `prototype`, `overrideHeight`, `height`, and `heightFrom` interact with the technique system.
