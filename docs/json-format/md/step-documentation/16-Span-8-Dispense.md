# Span-8 Dispense

| Property | Value |
|----------|-------|
| stepType | `"Span-8 Dispense"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7 |

---

## Behavior Summary

The Span-8 Dispense step delivers liquid from the tips of a Span-8 pod into destination labware. It shares its implementation with Span-8 Aspirate, distinguished by `aspirate` being `false`. At enqueue time the step validates the pod type (must implement `IPodS8`), verifies that all selected probes have tips of the same type and syringe size, resolves the target labware position and first well, and builds one or more group collections based on the selected technique. When `emptyTips` is `true`, the step ignores the `amount`/`amounts` values and instead dispenses the entire current contents of each tip (the volume is determined dynamically from the tip contents at runtime). If `discardExcess` is `true`, after the dispense completes a wash step is enqueued that discards any liquid remaining in the tips to waste. The step sets the pod's valve state to output and records `"Dispense"` as the last operation on the pod. For liquid type resolution, `"Tip Contents"` is the standard convention for dispense (it resolves the actual liquid type from the last non-air content in the tips), while `"Well Contents"` is invalid for dispense steps.

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `aspirate` | boolean | — | No | `false` | Must be `false` for this step type. Distinguishes Span-8 Dispense from Span-8 Aspirate. **Omission defaults to `true` (aspirate behavior)**, so this key must be explicitly set to `false` for correct Span-8 Dispense operation. |
| `pod` | string | — | Yes | `""` | Name of the Span-8 pod to use (e.g., `"Pod1"` on a standalone Span-8 — i5-Span / i7-Span; `"Pod2"` on an i7 Hybrid). Must resolve to a Span-8 pod. |
| `podType` | string | — | No | `"Span8"` | Pod-family label — always `"Span8"` for this step type. **If you write it, keep it correct**: an out-of-sync value misrepresents the step wherever the label is shown. It does not select the pod family — that comes from the named `pod` — and omitting it does not fail the step, so it is not author-required. Every editor-produced step carries `podType: "Span8"`. |
| `where` | string | — | Yes | `""` | Deck position or labware name of the destination. Supports expressions (prefix with `=`). |
| `what` | string | — | Yes | `""` | Labware class name of the destination labware (e.g., `"BCFlat96"`). Must be a pipettable labware class. |
| `useProbes` | comArray (boolean[8]) | — | No | All `true` | Array of eight booleans indicating which probes (1–8) are active. Serialized as `{"_biomekType": "comArray", "arraySubtype": "boolean", "values": [...]}`. Omission does not throw (consistent with the Span-8 Load/Unload/Wash docs). |
| `amount` | string | µL | Conditional | `""` | Volume to dispense, uniform across all selected probes. Supports expressions (prefix with `=`). Required unless `amounts` is specified or `emptyTips` is `true`. |
| `amounts` | comArray (string[8]) | µL | Conditional | Not present | Array of eight volume strings, one per probe, for individual probe volumes. Overrides `amount` when present. Supports expressions. Serialized as a comArray envelope — `{"_biomekType": "comArray", "arraySubtype": "string", "values": [...]}` — a SafeArray of strings, the same wrapper `useProbes` uses (there with `arraySubtype: "boolean"`). |
| `spacing` | string | wells | No | `""` | Number of wells between probes. Determines how far apart selected probes address wells in the labware. Supports expressions. When empty, the step substitutes the labware's minimum interval (labware-dependent: 1 for 96-well, 2 for 384-well). |
| `firstWell` | integer | — | Yes | `-1` (unconfigured) | 1-based numeric index of the first well to access (the well assigned to the lowest-numbered active probe). `-1` is only the truly-unconfigured sentinel (no labware picked yet; enqueue raises a "no well selected" error). Once a labware is configured the editor resolves it to `1`; a configured method carries a valid 1-based index (or uses the well expression). |
| `firstWellExpression` | string | — | No | `""` | Expression or well coordinate string used to calculate the first well when `useWellExpression` is `true`. Supports alphanumeric well references (e.g., `"A1"`) and expressions (e.g., `"=col"`). |
| `useWellExpression` | boolean | — | No | `false` | When `true`, the step uses `firstWellExpression` instead of `firstWell` to determine the starting well. Omission defaults to `false`. |
| `liquidtype` | string | — | Yes | `"Tip Contents"` | The liquid type for technique selection. May be a named liquid or `"Tip Contents"` (auto-detect from tip contents). `"Well Contents"` is **invalid** for dispense steps. Serialized all-lowercase (`liquidtype`) in method exports; case-insensitive import means either casing works, but author the lowercase form to match real exports. |
| `emptyTips` | boolean | — | No | `false` | When `true`, the step dispenses the entire current contents of each tip, ignoring `amount`/`amounts`. |
| `discardExcess` | boolean | — | No | `false` | When `true`, after the dispense any remaining liquid in the tips is discarded to waste via a wash step. |
| `autoSelectPrototype` | boolean | — | Yes | — | When `true`, the technique is auto-selected based on liquid type, labware, tip type, and volume. |
| `prototype` | string | — | Conditional | `""` | Name of the pipetting technique (transfer prototype) to use. Required when `autoSelectPrototype` is `false`. |
| `customPrototype` | object (`_biomekType:"technique"`) | — | No | *(absent)* | Inline custom pipetting technique object. When present, takes precedence over `prototype`. Omission is safe. |
| `overrideHeight` | boolean | — | Yes | `false` | When `true`, the step overrides the technique's default tip height with the values in `height` and `heightFrom`. |
| `height` | number or expression-capable string | mm | Conditional | `0.0` | Tip height offset from the reference point when `overrideHeight` is `true`. May be negative. Expression-capable (prefix with `=`). |
| `heightFrom` | integer or string | — | Conditional | `0` | Reference point for height measurement: `0` or `"Liquid"` = liquid surface, `1` or `"Bottom"` = well bottom, `2` or `"Top"` = well top. String aliases are case-insensitive. Expression-capable (prefix with `=`). |
| `customHeight` | boolean | — | No | `false` | Encoding discriminator for `height`/`heightFrom`. When `false`, those keys carry numeric/enumerated values; when `true`, they carry expression text. The editor writes it on every save. Set it to match the form of `height` you author. |
| `operation` | string | — | No | `"Dispense"` | Operation identifier saved by the technique selector. Always `"Dispense"` for this step type. |
| `useExpression` | boolean | — | No | `false` | When `true`, the probe selection is determined by `mandrelExpression` instead of `useProbes`. |
| `mandrelExpression` | string | — | Conditional | `""` | Expression that evaluates to the probe selection at runtime. Used when `useExpression` is `true`. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

> **Well numbering.** Wells are 1-based and **row-major**: on a 96-well plate `A1` = 1,
> `A12` = 12, `B1` = 13, …, `H12` = 96. See [Well Numbering](../Introduction-to-Pipetting-Steps.md#well-numbering).

---

## Enumerated / Constrained Values

| Parameter | Allowed Values | Notes |
|-----------|---------------|-------|
| `aspirate` | `false` | Must always be `false` for Span-8 Dispense |
| `podType` | `"Span8"` | Always `"Span8"` for this step type |
| `heightFrom` | `0`, `1`, `2` (or `"Liquid"`, `"Bottom"`, `"Top"`) | `0`/`"Liquid"` = from liquid surface, `1`/`"Bottom"` = from well bottom, `2`/`"Top"` = from well top. String aliases are case-insensitive. |
| `operation` | `"Dispense"` | Always `"Dispense"` for this step type |
| `liquidtype` | Any named liquid, `"Tip Contents"` | `"Well Contents"` is **invalid** for dispense steps |
| `useProbes` values | `true`, `false` | Exactly 8 elements; at least one must be `true` |
| `spacing` | Positive integer (as string), expression, or `""` | If empty, uses labware minimum interval. If provided, minimum depends on labware geometry (e.g., minimum 2 for 384-well plates). Supports expressions. |

---

## Cross-Field Validation Rules

1. `aspirate` **must** be `false`. If `true`, the step behaves as Span-8 Aspirate.
2. `podType` is **not author-required** (see §Parameters row) and is safely omitted from
   authored JSON. The runtime pod-family check is on the *resolved `pod`* (must be a Span-8 pod), not on
   this key — the step writes `"Span8"` itself at construction, so exports may carry it but
   authoring is free to leave it out.
3. If `autoSelectPrototype` is `false`, then `prototype` **must** be a non-empty string naming a valid pipetting technique.
4. If `overrideHeight` is `true`, then `height` and `heightFrom` **must** be provided with valid values.
5. If `useWellExpression` is `true`, then `firstWellExpression` **must** be a non-empty string. If `false`, `firstWell` must be a valid 1-based well index.
6. If `useExpression` is `true`, then `mandrelExpression` **must** be a non-empty expression string. If `false`, `useProbes` must be present with at least one `true` element.
7. `amounts` and `amount` are mutually exclusive: if `amounts` is present, `amount` is ignored. One or the other must provide a valid volume — unless `emptyTips` is `true`, in which case both are ignored.
8. All selected probes (those with `true` in `useProbes`) must have the **same tip type** and the **same syringe size** at runtime.
9. `liquidtype` **must not** be `"Well Contents"` for a dispense step; using it raises a "cannot be used" error. Use `"Tip Contents"` or a named liquid type.
10. When `emptyTips` is `true`, `amount` may be empty (`""`) or `"0"`; `amounts` may be absent. The runtime determines volumes dynamically from tip contents.
11. `discardExcess` is only meaningful for dispense steps. When `true`, a wash step is enqueued after the dispense that discards remaining tip contents to waste.
12. `spacing` must be compatible with the labware geometry — e.g., 384-well plates require a minimum spacing of 2. An empty string is valid (substitutes the labware's minimum interval).
13. When `emptyTips` is `true` and `autoSelectPrototype` is `true`, each probe may get a different technique based on its individual tip volume.
14. The dispense volume (`amount` or a per-probe `amounts` entry) **must not** exceed the tip contents: dispensing more than a tip holds raises a "dispense-too-much" error.
15. The dispensed volume **must not** overflow the destination well: exceeding well capacity raises a well-overflow error. Both checks apply per-probe for `amounts`.
16. When `emptyTips` is `true`, the step ignores `amount`/`amounts` **and** `discardExcess` is moot: the editor disables Discard Excess, Volume, Individual Volumes, and Edit... whenever Empty Tips is checked. A method that sets both `emptyTips: true` and `discardExcess: true` is contradictory — `emptyTips` wins and the tips are already emptied, so the discard is redundant.

---

## Structural Context

This step is a leaf and does not support subSteps.

---

## Canonical Examples

> The examples use `"pod": "Pod1"`. The Span-8 pod is `Pod1` on a standalone Span-8 instrument
> (i5-Span, i7-Span) and `Pod2` (the second/rightmost pod) on an i7 Hybrid. `Pod1`
> is valid on a standalone Span-8. Use whichever pod your
> instrument defines; an unresolved name raises an "unknown pod" error.

### Example 1: Basic Span-8 Dispense to a 96-well plate (all 8 probes, specified technique)

```json
{
  "stepType": "Span-8 Dispense",
  "parameters": {
    "amount": "5",
    "aspirate": false,
    "autoSelectPrototype": false,
    "customHeight": false,
    "discardExcess": false,
    "emptyTips": false,
    "firstWell": 9,
    "firstWellExpression": "=col",
    "height": 0.0,
    "heightFrom": 2,
    "liquidtype": "Tip Contents",
    "mandrelExpression": "",
    "operation": "Dispense",
    "overrideHeight": false,
    "pod": "Pod1",
    "prototype": "S8 MultiDispense",
    "spacing": "1",
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

### Example 2: Empty Tips dispense (dispense all tip contents)

```json
{
  "stepType": "Span-8 Dispense",
  "parameters": {
    "amount": "",
    "aspirate": false,
    "autoSelectPrototype": false,
    "customHeight": false,
    "discardExcess": false,
    "dynamic?": true,
    "emptyTips": true,
    "firstWell": 7,
    "firstWellExpression": "",
    "height": 0.0,
    "heightFrom": 2,
    "liquidtype": "Tip Contents",
    "mandrelExpression": "",
    "operation": "Dispense",
    "overrideHeight": false,
    "pod": "Pod1",
    "prototype": "S8 MultiDispense",
    "spacing": "1",
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

### Example 3: Dispense with height override, expression-based well, and subset of probes

```json
{
  "stepType": "Span-8 Dispense",
  "parameters": {
    "amount": "10",
    "aspirate": false,
    "autoSelectPrototype": true,
    "customHeight": false,
    "discardExcess": false,
    "emptyTips": false,
    "firstWell": 1,
    "firstWellExpression": "=col",
    "height": -2.0,
    "heightFrom": 0,
    "liquidtype": "Tip Contents",
    "mandrelExpression": "",
    "operation": "Dispense",
    "overrideHeight": true,
    "pod": "Pod1",
    "prototype": "",
    "spacing": "1",
    "useExpression": false,
    "useProbes": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true, true, true, true, false, false, false, false]
    },
    "useWellExpression": true,
    "what": "BCFlat96",
    "where": "P5"
  }
}
```

---

## Common Mistakes

- **`"Well Contents"` as `liquidtype`**: raises a "cannot be used" error. Use `"Tip Contents"` or a named liquid.
- **Mixing probe tip types or syringe sizes across the selected probes**: the runtime rejects the mismatch — a probe with a fixed tip and a probe with a disposable tip cannot be co-selected.
- **`spacing` too small for the labware**: 384-well plates need `spacing >= 2`. `""` (empty string) is fine — the step substitutes the labware's minimum interval.
- **Confusing `useExpression` (probes) with `useWellExpression` (well)**: independent flags. The serialized well-by-expression key is `useWellExpression`, not `firstWellByExpression`.
- **Authoring `refreshTips`/`tipLabwareClass`**: aspirate-only. Not written by the span-8 dispense step and should be omitted.

---

## Differences from Span-8 Aspirate

| Aspect | Span-8 Aspirate | Span-8 Dispense |
|--------|----------------|-----------------|
| `aspirate` key | `true` | `false` |
| `operation` key | `"Aspirate"` | `"Dispense"` |
| `liquidtype` default | `"Well Contents"` | `"Tip Contents"` |
| `liquidtype` restrictions | `"Tip Contents"` is invalid | `"Well Contents"` is invalid |
| `emptyTips` | Not applicable (should be `false`) | Dispense all tip contents when `true` |
| `discardExcess` | Not applicable (should be `false`) | Discard remaining tip contents after dispense |
| `refreshTips` / `tipLabwareClass` | Available (load fresh tips before aspiration) | Not available (not set at creation) |
| Post-operation behavior | Records `"Aspirate"` as last operation | Records `"Dispense"` as last operation; if next operation is aspirate, a wash is auto-inserted |

---

## Related Guides

- **[Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md)** — Technique auto-selection algorithm, height override mechanics, `C__` pipetting variable reference, and template architecture. Covers how `autoSelectPrototype`, `prototype`, `overrideHeight`, `height`, and `heightFrom` interact with the technique system.
- **Mixing in place** — mixing at a well is a technique property. Configure it on the Span-8 technique's **Mix** tab in the Technique Editor (mix volume, cycles, aspirate/dispense heights and speeds, follow-liquid-level, touch-tips, leading-air-gap-and-blowout). An item-level `mixCount` is inert on every pod family and Span-8 Dispense does not read it. See [Transfer → Mixing in place](11-Transfer.md#mixing-in-place).
