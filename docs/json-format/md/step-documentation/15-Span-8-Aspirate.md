# Span-8 Aspirate

| Property | Value |
|----------|-------|
| stepType | `"Span-8 Aspirate"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7 |

---

## Behavior Summary

The Span-8 Aspirate step removes liquid from source labware using one or more probes on a Span-8 pod. It provides direct control over individual probe aspiration volumes and well access.

At enqueue time the step validates pod type (must be a Span-8 pod), verifies selected probes have tips of the same type and syringe size, resolves the target labware position and first well, and optionally refreshes tips. If tips were last used for a dispense, a wash step is automatically inserted before aspiration to discard residual liquid.

The step shares its implementation with Span-8 Dispense (the `aspirate` boolean key distinguishes the two).

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `aspirate` | boolean | — | No | `true` | Must be `true` for this step type. Distinguishes Span-8 Aspirate from Span-8 Dispense (which sets this to `false`). Omission defaults to `true` (correct for aspirate), but author explicitly for clarity and to distinguish from Span-8 Dispense. |
| `pod` | string | — | Yes | `""` | Name of the Span-8 pod to use (e.g., `"Pod1"` on a standalone Span-8 — i5-Span / i7-Span; `"Pod2"` on an i7 Hybrid). Must resolve to a Span-8 pod; an unresolved name raises an "unknown pod" error, and a non-Span-8 pod raises a "wrong pod type" error. |
| `podType` | string | — | No | `"Span8"` | Pod-family label — always `"Span8"` for this step type. **If you write it, keep it correct**: an out-of-sync value misrepresents the step wherever the label is shown. It does not select the pod family — that comes from the named `pod` — and omitting it does not fail the step, so it is not author-required. Every editor-produced step carries `podType: "Span8"`. |
| `where` | string | — | Yes | `""` | Deck position or labware name where the source labware is located. Supports expressions (prefix with `=`). |
| `what` | string | — | Yes | `""` | Labware class name of the source labware (e.g., `"BCFlat96"`). Must be a pipettable labware class. This field uses the short labware class name; the `LabwareClasses\\` path prefix is used only in Instrument Setup's `deckItems`. |
| `useProbes` | comArray (boolean[8]) | — | No | All `true` | Array of eight booleans indicating which probes (1–8) are active. Serialized as `{"_biomekType": "comArray", "arraySubtype": "boolean", "values": [...]}`. Omission does not throw (consistent with the Span-8 Load/Unload/Wash docs). |
| `amount` | string | µL | Conditional | `""` | Volume to aspirate, uniform across all selected probes. Supports expressions. Required unless `amounts` is specified. |
| `amounts` | comArray (string[8]) | µL | Conditional | Not present | Array of eight volume strings, one per probe, for individual probe volumes. Overrides `amount` when present. Supports expressions. Serialized as a comArray envelope — `{"_biomekType": "comArray", "arraySubtype": "string", "values": [...]}` — a SafeArray of strings. |
| `spacing` | string | wells | No | `""` | Number of wells between probes. Determines how far apart selected probes address wells in the labware. Supports expressions. When empty, the step substitutes the labware's minimum interval (labware-dependent: 1 for 96-well, 2 for 384-well). |
| `firstWell` | integer | — | Yes | `-1` (unconfigured) | 1-based numeric index of the first well to access (the well assigned to the lowest-numbered active probe). `-1` is only the truly-unconfigured sentinel (no labware picked yet; enqueue raises a "no well selected" error). Once a labware is configured the editor resolves it to `1`; a configured method carries a valid 1-based index (or uses the well expression). |
| `firstWellExpression` | string | — | No | `""` | Expression or well coordinate string used to calculate the first well when `useWellExpression` is `true`. Supports alphanumeric well references (e.g., `"A1"`) and expressions (e.g., `"=col"`). |
| `useWellExpression` | boolean | — | No | `false` | When `true`, the step uses `firstWellExpression` instead of `firstWell` to determine the starting well. Omission defaults to `false`. |
| `liquidtype` | string | — | Yes | `"Well Contents"` | The liquid type for technique selection. May be a named liquid, `"Well Contents"` (auto-detect from labware), or `"Tip Contents"` (invalid for aspirate). Serialized all-lowercase (`liquidtype`) in method exports; case-insensitive import means either casing works, but author the lowercase form to match real exports. |
| `refreshTips` | boolean | — | No | `false` | When `true`, discards current tips and loads fresh tips from the specified tip labware class before aspirating. |
| `tipLabwareClass` | string | — | Conditional | `""` | Labware class name of tips to load when `refreshTips` is `true` (e.g., `"BC230"`). |
| `autoSelectPrototype` | boolean | — | Yes | — | When `true`, the technique is auto-selected based on liquid type, labware, tip type, and volume. |
| `prototype` | string | — | Conditional | `""` | Name of the pipetting technique to use. Required when `autoSelectPrototype` is `false`. |
| `customPrototype` | object (`_biomekType:"technique"`) | — | No | *(absent)* | Inline custom pipetting technique object. When present, takes precedence over `prototype`. Omission is safe. |
| `overrideHeight` | boolean | — | No | `false` | When `true`, the step overrides the technique's default tip height with the values in `height` and `heightFrom`. Omission defaults to `false` (no height override). |
| `height` | number or expression-capable string | mm | Conditional | `0.0` | Tip height offset from the reference point when `overrideHeight` is `true`. May be negative (e.g., `-2.0` means 2 mm below the reference). Expression-capable. |
| `heightFrom` | integer or string | — | Conditional | `0` | Reference point for height measurement: `0` or `"Liquid"` = liquid surface, `1` or `"Bottom"` = well bottom, `2` or `"Top"` = well top. String aliases are case-insensitive. Expression-capable (prefix with `=`). |
| `customHeight` | boolean | — | No | `false` | Encoding discriminator for `height`/`heightFrom`. When `false`, those keys carry numeric/enumerated values; when `true`, they carry expression text. The editor writes it on every save. Set it to match the form of `height` you author. |
| `operation` | string | — | No | `"Aspirate"` | Operation identifier saved by the technique selector. Always `"Aspirate"` for this step type. |
| `useExpression` | boolean | — | No | `false` | When `true`, the probe selection is determined by `mandrelExpression` instead of `useProbes`. |
| `mandrelExpression` | string | — | Conditional | `""` | Expression that evaluates to the probe selection at runtime. Used when `useExpression` is `true`. |
| `emptyTips` | boolean | — | No | `false` | Not applicable to aspirate (used by dispense). Should be `false`. |
| `discardExcess` | boolean | — | No | `false` | Not applicable to aspirate (used by dispense). Should be `false`. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md). Key matching is case-insensitive (§1), so the `stepUI`/`stepui` casing seen in exports imports either way; author the first-letter-lowercase form._

> **Well numbering.** Wells are 1-based and **row-major**: on a 96-well plate `A1` = 1,
> `A12` = 12, `B1` = 13, …, `H12` = 96. See [Well Numbering](../Introduction-to-Pipetting-Steps.md#well-numbering).

---

## Enumerated / Constrained Values

| Parameter | Allowed Values | Notes |
|-----------|---------------|-------|
| `aspirate` | `true` | Must always be `true` for Span-8 Aspirate |
| `podType` | `"Span8"` | Always `"Span8"` for this step type |
| `heightFrom` | `0`, `1`, `2` (or `"Liquid"`, `"Bottom"`, `"Top"`) | `0`/`"Liquid"` = from liquid surface, `1`/`"Bottom"` = from well bottom, `2`/`"Top"` = from well top. String aliases are case-insensitive. |
| `operation` | `"Aspirate"` | Always `"Aspirate"` for this step type |
| `liquidtype` | Any named liquid, `"Well Contents"` | `"Tip Contents"` is **invalid** for aspirate steps |
| `useProbes` values | `true`, `false` | Exactly 8 elements; at least one must be `true` |
| `spacing` | Positive integer (as string), expression, or `""` | If empty, uses labware minimum interval. If provided, minimum depends on labware geometry (e.g., minimum 2 for 384-well plates). Supports expressions. |

---

## Cross-Field Validation Rules

1. `aspirate` **must** be `true`. If `false`, the step behaves as Span-8 Dispense.
2. The named `pod` **must** resolve to a Span-8 pod. An unresolved name raises an "unknown pod" error; a pod that is not Span-8 raises a "wrong pod type" error. `podType` is written as `"Span8"` by the step itself and is not the source of pod-family resolution.
3. If `refreshTips` is `true`, then `tipLabwareClass` **must** be a non-empty string naming a valid tip labware class.
4. If `autoSelectPrototype` is `false`, then `prototype` **must** be a non-empty string naming a valid pipetting technique.
5. If `overrideHeight` is `true`, then `height` and `heightFrom` **must** be provided with valid values.
6. If `useWellExpression` is `true`, then `firstWellExpression` **must** be a non-empty string. If `false`, `firstWell` must be a valid 1-based well index.
7. If `useExpression` is `true`, then `mandrelExpression` **must** be a non-empty expression string. If `false`, `useProbes` must be present with at least one `true` element.
8. `amounts` and `amount` are mutually exclusive: if `amounts` is present, `amount` is ignored. One or the other must provide a valid volume.
9. All selected probes (those with `true` in `useProbes`) must have the **same tip type** and the **same syringe size** at runtime.
10. `liquidtype` **must not** be `"Tip Contents"` for an aspirate step — using it raises a "cannot be used" error. `"Well Contents"` is valid and means the liquid type is auto-detected from the labware data.
11. `emptyTips` should be `false` for aspirate steps; setting it `true` has no meaningful effect on aspiration.
12. `discardExcess` should be `false` for aspirate steps; it is only meaningful for dispense.
13. `spacing` must be compatible with the labware geometry — e.g., 384-well plates require a minimum spacing of 2. An empty string is valid (substitutes the labware's minimum interval).

---

## Structural Context

This step is a leaf and does not support subSteps.

---

## Canonical Examples

> The Span-8 pod is `Pod1` on a standalone Span-8 instrument (i5-Span, i7-Span) and `Pod2`
> (the second/rightmost pod) on an i7 Hybrid. The examples below use `Pod1` because it is valid on
> any Span-8-equipped instrument — use whichever pod your instrument defines.

### Example 1: Basic Span-8 Aspirate from a 96-well plate (all 8 probes, auto-select technique)

```json
{
  "stepType": "Span-8 Aspirate",
  "parameters": {
    "amount": "10",
    "aspirate": true,
    "autoSelectPrototype": true,
    "customHeight": false,
    "discardExcess": false,
    "emptyTips": false,
    "firstWell": 1,
    "firstWellExpression": "",
    "height": 0.0,
    "heightFrom": 0,
    "liquidtype": "Well Contents",
    "mandrelExpression": "",
    "operation": "Aspirate",
    "overrideHeight": false,
    "pod": "Pod1",
    "prototype": "",
    "refreshTips": false,
    "spacing": "1",
    "tipLabwareClass": "",
    "useExpression": false,
    "useProbes": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true, true, true, true, true, true, true, true]
    },
    "useWellExpression": false,
    "what": "BCFlat96",
    "where": "P13"
  }
}
```

### Example 2: Aspirate with specific technique, height override, tip refresh, and expression-based first well

```json
{
  "stepType": "Span-8 Aspirate",
  "parameters": {
    "amount": "=aspirateVolume",
    "aspirate": true,
    "autoSelectPrototype": false,
    "customHeight": false,
    "discardExcess": false,
    "emptyTips": false,
    "firstWell": 1,
    "firstWellExpression": "=col",
    "height": -2.0,
    "heightFrom": 0,
    "liquidtype": "Water",
    "mandrelExpression": "",
    "operation": "Aspirate",
    "overrideHeight": true,
    "pod": "Pod1",
    "prototype": "S8 1000 High",
    "refreshTips": true,
    "spacing": "1",
    "tipLabwareClass": "BC230",
    "useExpression": false,
    "useProbes": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true, true, true, true, true, true, true, true]
    },
    "useWellExpression": true,
    "what": "BCFlat96",
    "where": "P13"
  }
}
```

### Example 3: Aspirate with individual probe volumes and subset of probes

```json
{
  "stepType": "Span-8 Aspirate",
  "parameters": {
    "amounts": {
      "_biomekType": "comArray",
      "arraySubtype": "string",
      "values": ["10", "10", "0", "0", "0", "0", "0", "0"]
    },
    "aspirate": true,
    "autoSelectPrototype": false,
    "customHeight": false,
    "discardExcess": false,
    "emptyTips": false,
    "firstWell": 5,
    "firstWellExpression": "",
    "height": 1.0,
    "heightFrom": 1,
    "liquidtype": "Serum",
    "mandrelExpression": "",
    "operation": "Aspirate",
    "overrideHeight": true,
    "pod": "Pod1",
    "prototype": "S8 1000 Low",
    "refreshTips": false,
    "spacing": "2",
    "tipLabwareClass": "",
    "useExpression": false,
    "useProbes": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true, true, false, false, false, false, false, false]
    },
    "useWellExpression": false,
    "what": "BCFlat96",
    "where": "P5"
  }
}
```

---

## Common Mistakes

- **`"Tip Contents"` as `liquidtype`**: raises a "cannot be used" error. Use `"Well Contents"` or a named liquid.
- **Mixing probe tip types or syringe sizes across the selected probes**: the runtime rejects the mismatch. Applies at the pod, not the step — a probe with a fixed tip and a probe with a disposable tip cannot be co-selected.
- **`spacing` too small for the labware**: 384-well plates need `spacing >= 2`. `""` (empty string) is fine — the step substitutes the labware's minimum interval.
- **Confusing `useExpression` (probes) with `useWellExpression` (well)**: independent flags. Similarly, the serialized key for the well-by-expression flag is `useWellExpression`, not `firstWellByExpression`.
- **Providing both `amounts` and `amount`**: `amounts` takes precedence; `amount` is silently ignored. Pick one.

---

## Related Guides

- **[Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md)** — Technique auto-selection algorithm, height override mechanics, `C__` pipetting variable reference, and template architecture. Covers how `autoSelectPrototype`, `prototype`, `overrideHeight`, `height`, and `heightFrom` interact with the technique system.
- **Mixing in place** — mixing at a well is a technique property. Configure it on the Span-8 technique's **Mix** tab in the Technique Editor (mix volume, cycles, aspirate/dispense heights and speeds, follow-liquid-level, touch-tips, leading-air-gap-and-blowout). An item-level `mixCount` is inert on every pod family and Span-8 Aspirate does not read it. See [Transfer → Mixing in place](11-Transfer.md#mixing-in-place).
