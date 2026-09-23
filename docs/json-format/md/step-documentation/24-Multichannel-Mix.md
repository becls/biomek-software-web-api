# Multichannel Mix

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Mix"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5 or i7 with a Multichannel pod |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Universal base keys (`caption`, `dynamic?`, `disabled`, …) are **not** re-documented here.

---

## Behavior Summary

The Multichannel Mix step aspirates and re-dispenses liquid in place, `mixCount`
times, to mix well contents using a fixed multichannel head (96- or
384-channel). It shares behavior with Multichannel Aspirate and Dispense,
differing only in that it defaults `operation` to `"Mix"` and adds a `mixCount`
property. At enqueue time the step validates that the pod is a multichannel pod
(a Span-8 or Fixed-8 pod produces an error), resolves the deck position, verifies labware is present and on top of any
stack, checks labware-class compatibility and the `"Can Pipette"`
characteristic, and confirms tips are loaded. If `refreshTips` is `true`,
current tips are discarded and fresh tips loaded from `tipLabwareClass`. It then
uses the given pipetting technique to mix the given volume the given number of
times in the selected wells. The number of mix cycles and the per-cycle mix volume
are driven by `mixCount` and `volume` through the technique's mix action.

Note that Multichannel Aspirate, Multichannel Dispense, Multichannel Mix, and
Multichannel Wash Tips share much of their implementation. As a result, some
parameters are listed here that are only used by those other steps. See each
step's documentation for details.

---

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `operation` | string | — | Yes | `"Mix"` | Step action identity. Must be `"Mix"`. |
| `pod` | string | — | Yes | `""` | Name of the multichannel pod to use (e.g., `"Pod1"`). Must resolve to a multichannel pod. The MC pod is `"Pod1"` on a standalone Multichannel instrument (i5-MC, i7-MC) and on an i7 Hybrid; it is `"Pod2"` only when the MC is the right-hand pod of a dual-pod i7. Never author `"MC"`/`"MPod"`. See [Introduction to Biomek Method JSON](../Introduction-to-Biomek-Method-JSON.md#instrument-variants) §"Instrument Variants" / "Pod Types". |
| `location` | string | — | Yes | `""` | Deck position or labware name where the source labware is located. Supports expressions (prefix with `=`). Accepts a deck-position name (e.g. `P5`) or a labware **instance** name (assigned via the labware's `properties.name` in Instrument Setup) — matched case-insensitively. See [Instrument Setup](03-Instrument-Setup.md) and [Labware and Position Resolution](../concept-guides/02-labware-and-position-resolution.md). |
| `labwareClass` | string | — | Yes | `""` | Expected labware class at the position (e.g., `"BCFlat96"`). Must be compatible with the actual labware. This field uses the short labware class name; the `LabwareClasses\\` path prefix is used only in Instrument Setup's `deckItems`. |
| `liquidtype` | string | — | Yes | `""` | Liquid classification for technique selection. Commonly `"Well Contents"` (auto-detect from wells) or a named liquid type. |
| `volume` | number or string | µL | Yes | `0.0` | Volume aspirated/dispensed per mix cycle. Positive numeric value or expression. |
| `mixCount` | expression-capable string | — | Yes | `"1"` | Number of mix cycles. Positive integer value or expression. |
| `selectionInfo` | comArray (`arraySubtype: "integer"`) | — | Conditional | `""` (empty) | Array of 1-based section/quadrant indices to mix in. On these low-level Multichannel steps `selectionInfo` is a `comArray` of `arraySubtype: "integer"` carrying section/quadrant indices (e.g. `[1]`) — **different** from the item-level `selectionInfo` on Transfer/Combine, which is a well selection whose subtype varies by pod (boolean well-mask on Fixed-8, integer on Span-8/MC). **Must contain exactly one element** — the step requires exactly one target group per invocation (see Cross-Field rule 10); to process multiple sections, use a Loop with `sectionExpression` or multiple Multichannel Mix steps. Required when `useExpression` is `false`. Serialized as `{"_biomekType": "comArray", "arraySubtype": "integer", "values": [...]}`.|
| `useExpression` | boolean | — | No | `false` | When `true`, selection comes from `sectionExpression` instead of `selectionInfo`. |
| `sectionExpression` | string | — | Conditional | `""` | Expression used for section selection at runtime. Required when `useExpression` is `true`. |
| `refreshTips` | boolean | — | Yes | `false` | Whether to discard current tips and load fresh tips before mixing. |
| `tipLabwareClass` | string | — | Conditional | `""` | The tips to use, specified either as a labware class, a labware instance name, or a position name. Expression capable. Required when `refreshTips` is `true`. |
| `autoSelectPrototype` | boolean | — | Yes | `false` | Whether the software auto-selects a technique based on pod type, tip type, labware, liquid type, and volume. |
| `prototype` | string | — | Conditional | `""` | Explicit technique name. Required when `autoSelectPrototype` is `false` and `customPrototype` is absent. |
| `customPrototype` | object (`_biomekType: "technique"`) | — | No | Not present | Inline technique object (`_biomekType: "technique"` payload). When present, overrides both `prototype` and `autoSelectPrototype`. |
| `overrideHeight` | boolean | — | Yes | `false` | **Inactive on Mix**. Mix steps do not allow overriding technique height. However, this key must be present. Author it as `false`. |
| `height` | number or string | mm | No | `0.0` | **Inactive on Mix**. Do not author this key. |
| `heightFrom` | integer or string | — | No | `0` (long) | **Inactive on Mix**. Do not author this key. |

> **A liquid-relative height needs a locatable liquid surface.** With a technique that follows the liquid, the target labware's surface must be computable at run time: its `volumeType` — set in Instrument Setup / Guided Setup — must be `"Nominal"` or `"Known"` **with** an `evalAmounts` fill. Multichannel (96-/384-channel) and Fixed-8 pods do **not** support liquid-level sensing, so on those pods an `"Unknown"` well cannot be rescued at run time — the move throws. See [Pipetting Techniques and Templates](../Pipetting-Techniques-and-Templates.md#liquid-types) §"Liquid types" / §"Liquid-relative heights need a locatable liquid surface".

### Editor-persisted keys (not read by the Mix engine)

These are written on every export by the editor but are **not** read at enqueue. Include them to match editor output; classify as **inert UI-state**.

| Key | Type | Units | Default | Description / classification |
|-----|------|-------|---------|------------------------------|
| `customHeight` | boolean | — | `false` | Encoding discriminator for `height`: `false` = numeric offset; `true` = text/custom height, which also implies `overrideHeight`. For Mix the height panel is disabled, so this is effectively always `false`. See [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#height-encoding-keys). |
| `pattern` | comArray (`arraySubtype: "boolean"`) | — | (well-count mask) | **Inert UI-state.** Boolean well-selection mask maintained by the labware view to restore the visual selection; the engine uses `selectionInfo`, not `pattern`. In real exports this is a full-length boolean array matching the labware's well count. |
| `aspirateTime` | expression-capable string | s | `"1"` | **Inactive and obsolete.** This key is not used. |
| `washSettleTime` | expression-capable string | s | `"0"` | **Inactive and obsolete.** This key is not used. |

---

## Enumerated / Constrained Values

| Parameter | Allowed Values | Notes |
|-----------|---------------|-------|
| `operation` | `"Mix"` | Must always be `"Mix"` for this step type. |
| `heightFrom` | `0`, `1`, `2` (or `"Liquid"`, `"Bottom"`, `"Top"`) | Inactive on Mix. |
| `liquidtype` | Any named liquid, `"Well Contents"` | `"Well Contents"` is the editor default for Mix. `"Tip Contents"` is **invalid** for Mix (only valid for Dispense), raising a runtime error. |
| `selectionInfo` values | Positive integers (1-based) | The array iterates the *sections* the head touches, so the valid values match how many sections the labware exposes to the head — **not** the plate's column count. For a **96-well plate + 96-channel head** there is exactly one section, so only `[1]` is valid; authoring `[1,2,3]` fails because the extra indices do not match sections the head exposes on that labware. For a **384-well plate + 96-channel head** or **1536-well plate + 384-channel head** each labware exposes four interleaved quadrants and the valid values are quadrant indices `1`–`4`. However, the Multichannel Mix step requires that there be only a single section used, so this array must have exactly one value. |

---

## Cross-Field Validation Rules

1. `operation` **must** equal `"Mix"`. The step defaults to this value.
2. `pod` **must** resolve to a Multichannel pod. A Span-8 or Fixed-8 pod will produce a runtime error.
3. `location` **must** resolve to a valid, reachable deck position with labware present and on top of any stack.
4. `labwareClass` **must** be compatible with the actual labware class at the resolved position.
5. The labware class **must** have a `"Can Pipette"` characteristic, or the step
   raises a runtime error.
6. Tips **must** be loaded on the pod at runtime. If no tips are present, the step raises a no-tips error.
7. If `refreshTips` is `true`, `tipLabwareClass` **must** name a valid tip labware class.
8. If `autoSelectPrototype` is `false`, then `prototype` **must** be a non-empty string naming a valid technique. If the technique cannot be found, the step raises a technique-not-found error.
9. If `autoSelectPrototype` is `true`, at least one matching technique must exist for the given context (pod type, head type, tip class, labware, liquid type, volume), or the step raises a technique-selection error.
10. `selectionInfo` (or the result of `sectionExpression`) **must** resolve to
    exactly one valid target group (either 1 for a whole plate where the plate
    wells match the head geometry, or a quadrant if the plate has more wells
    than the head). Zero targets or multiple targets raise a
    selection error.
11. If `useExpression` is `true`, then `sectionExpression` **must** be a non-empty, evaluable expression string. If `false`, `selectionInfo` must be a non-empty array of valid section indices.
12. `liquidtype` **must not** be `"Tip Contents"` for a Mix step. When `"Well
    Contents"` is selected, all wells in the target group must share one liquid type at
    runtime, or a liquid-mismatch error is raised
13. `volume` (the per-cycle mix volume) runs through the same technique
   aspirate/dispense checks as the other MC steps: a volume greater than tip
    capacity raises a dispense-too-much error, and a volume that would overflow
    the well raises a well-overflow error. Additionally, `volume` must not
    exceed the well's tracked contents. See [Well Volume Tracking](../concept-guides/05-well-volume-tracking.md).
14. `mixCount` must resolve to a non-negative integer at runtime. A negative value raises a negative-mix-count error. A value of `0` is technically accepted and produces zero mix cycles.
15. Tips must be empty before a mix begins, or the step raises an error. Air gaps do not trip the check; anything classified as liquid does. Reusing tips that still hold liquid — a Multichannel Aspirate followed by a standalone Mix — is the common cause.

---

## Structural Context

This step is a leaf and does not support subSteps.

---

## Canonical Examples

### Example 1: Basic Multichannel Mix (auto-select technique), 3 cycles of 80 uL across all wells (section 1)

Includes the editor-persisted keys so it matches a real export.

```json
{
  "stepType": "Multichannel Mix",
  "parameters": {
    "operation": "Mix",
    "pod": "Pod1",
    "location": "P4",
    "labwareClass": "BCFlat96",
    "liquidtype": "Well Contents",
    "volume": 80,
    "mixCount": "3",
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
    "washSettleTime": "0"
  }
}
```

> The `pattern` array is shown empty for brevity; in a real export it is a full-length boolean mask matching the labware's well count. The engine ignores `pattern` and uses `selectionInfo`.

### Example 2: Explicit technique with tip refresh, expression-based selection

Engine-relevant keys only (the software supplies UI-state keys on next save).

```json
{
  "stepType": "Multichannel Mix",
  "parameters": {
    "operation": "Mix",
    "pod": "Pod1",
    "location": "P7",
    "labwareClass": "BCDeep96Square",
    "liquidtype": "Water",
    "volume": "=MixVol",
    "mixCount": "=MixCycles",
    "selectionInfo": {
      "_biomekType": "comArray",
      "arraySubtype": "integer",
      "values": []
    },
    "useExpression": true,
    "sectionExpression": "=CurrentQuadrant",
    "refreshTips": true,
    "tipLabwareClass": "BC230",
    "autoSelectPrototype": false,
    "prototype": "MC",
    "overrideHeight": false
  }
}
```

---

## Common Mistakes

- **Expecting `overrideHeight`/`height` to move the mix heights**: the Mix editor disables height editing, and mix aspirate/dispense heights come from the technique's Mix parameters, not the step-level. Leave `overrideHeight: false` — but the key must still be present, or enqueue throws.
- **Reading meaning into `aspirateTime` / `washSettleTime` / `pattern`**: these are inert UI-state, not engine inputs. `customHeight` is not in that group — it is the `height` encoding discriminator (see the parameter table) — but because Mix takes its heights from the technique, `false` is the only value worth authoring here.
- **`mixCount` of empty or negative**: `mixCount` must resolve to a non-negative integer at runtime. A negative value throws a negative-mix-count error; a value of `0` is technically accepted but results in no mixing (silent no-op).
- **Providing `selectionInfo` as a plain JSON array**: must be a `comArray` object (`arraySubtype: "integer"`).
- **Assuming `autoSelectPrototype` defaults to `true`**: default is `false`. Set it explicitly.

---

## Related Guides

- **[Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md)** — technique auto-selection, height override mechanics, the `C__Mix*` context variables, and `customHeight` vs `overrideHeight`.
- **[Method JSON Structure](../Method-JSON-Structure.md)** — envelope, `_biomekType`/`comArray` encoding, structural rules.
