# Span-8 Serial Dilution

| Property | Value |
|----------|-------|
| stepType | `"Span-8 Serial Dilution"` |
| Category | Leaf (expands into many enqueued child steps at runtime) |
| Terminator | N/A |
| Compatible Hardware | i5, i7 (Span-8 pod required) |

## Behavior Summary

The Span-8 Serial Dilution step performs a serial dilution across a contiguous run of
wells on a plate (or tube rack) using a Span-8 pod.

When enqueued, the step:
#. validates the step configuration, including:
  * the pod is a Span-8 pod
  * the dilution plate (or tube rack) resolves properly
  * the well selection pattern is contiguous along the chosen `direction`
  * the probe selection is valid

#. When the step is configured to pre-fill the plate with diluent from a reservoir (`prepareDiluentInPlate`),
  * it aspirates from the diluent reservoir and dispenses to the diluent labware
  * the diluent volume is computed to meet the ratio `1:dilutionRatio` given the dilution `volume`.
  * the diluent step uses `diluentAspirateProperties`/`diluentDispenseProperties` to control the pipetting behavior.
  * Tips are changed or washed per the distinct diluent tip handling settings.

#. It then walks the wells transferring `volume` µL of `liquidtype`
between adjacent wells along `direction`
  * Tips are changed or washed per the passive/active wash settings
  * Aspirate/dispense technique and height for the serial transfers come from the
    `aspirateProperties`/`dispenseProperties` sub-dictionaries;

#. When `transferToWaste` is true, discard the excess from the last wells to waste.

## Parameters Reference Table

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Universal base keys (`caption`, `defaultCaption`, `disabled`, …) live there and
> are **not** repeated here.
>
> Note: unlike the atomic Span-8 steps this step does **not**
> carry `dynamic?` at the top level.

**Key-naming decoder** (the cryptic `a`-prefix families):

* `a…` = tip cleaning settings for the *main* serial transfer
* `aDil…` = tip cleaning settings for the *diluent* transfer.

### Step-Level Keys — Position, labware, pod, volume

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | string (expression) | — | Yes | `""` | Span-8 pod name. Must resolve to a Span-8 pod. |
| `where` | string (expression) | — | Yes | `""` | Deck position of the dilution plate; validated at enqueue. |
| `position` | string | — | Yes | `""` | Deck position of the dilution plate. Must be the same value as `where`. |
| `what` | string | — | Yes | `""` | Labware **class** of the dilution plate; validated at enqueue. |
| `labwareClass` | string | — | Yes | `""` | Labware **class** of the dilution plate. Must be the same value as `what`. |
| `volume` | string (expression) | µL | Yes | `""` | Transfer volume per serial step. |
| `liquidtype` | string | — | Yes | `"Water"` | Liquid type for the serial transfers. **Serialized as `liquidtype` (all lowercase)** — differs from Fixed-8 steps which use `liquidType`; the runtime dictionary is case-insensitive. |
| `direction` | string (expression) | — | Yes | `"Left to Right"` | Dilution direction. Only `"Left to Right"` or `"Top to Bottom"` are legal. |
| `dilutionRatio` | string (expression) | ratio | Yes | `"2"` | Dilution ratio `1:N`; must be ≥ `1.0`. Diluent volume = `volume × (dilutionRatio − 1)`. |
| `wellsX` | integer | — | No | *(from labware)* | Number of columns of the dilution plate; the editor writes it from the labware class (`-1` if unknown). |

### Step-Level Keys — Tip handling

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `tips` | string | — | Yes | `""` | Selected tip type / labware class. |
| `tipLocation` | string | — | Conditional | `""` | Tip source used when loading fresh tips — a tip labware **class** (what the editor writes, e.g. `"BC230"`), a tip-box labware name, or a deck position. |
| `loadTips` | boolean | — | Yes | `true` | When `true`, the step auto-loads/refreshes tips as needed; when `false`, it requires tips already on the pod. |
| `useFixedTips` | boolean | — | No | `false` | Use the pod's fixed tips. Mutually exclusive with `useDisposableTips`/mandrel selection. |
| `useDisposableTips` | boolean | — | No | `false` | Use disposable tips on the pod. |
| `useMandrelSelection` | boolean | — | No | `false` | Inert UI-state: persists the editor's probe-selection mode (fixed / disposable / mandrel); not read by the enqueue engine. |
| `useCurrentTips` | boolean | — | No | `true` (editor writes `true`) | UI-written constant (`true`); the runtime tip lifecycle is decided per substep, not from this key. Treat as **inert-UI-state**. |
| `leaveTipsOn` | boolean | — | No | `true` (editor writes `true`) | UI-written constant (`true`). Inert at this level; the enqueued substeps set their own `LeaveTipsOn`. |

### Step-Level Keys — Probe & well selection

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `useProbes` | comArray (boolean[8]) | — | No | *(all true)* | Per-probe selection. `{"_biomekType":"comArray","arraySubtype":"boolean","values":[…]}`. Reads with a tolerant fallback to the all-probes-selected default, so omission does not throw (consistent with the Span-8 Load/Unload/Wash docs). |
| `mandrelExpression` | string | — | No | `""` | Expression-based probe selection. |
| `useMandrelExpression` | boolean | — | No | `false` | Authoritative flag: use `mandrelExpression` for probes. |
| `useSectionExpression` | boolean | — | No | `false` | Authoritative flag: use `sectionExpression` for **well** selection. |
| `useExpression` | boolean | — | No | `false` | **Scratch/overloaded key** — both the mandrel selector and the labware view persist to this same key (`UseExpression`), so its exported value is not authoritative. Prefer the two authoritative flags above; preserve `useExpression` verbatim on round-trip. |
| `sectionExpression` | string | — | No | `""` | Well-selection expression. Alternative to `selectionInfo`/`pattern` when `useSectionExpression: true`. |
| `selectionInfo` | comArray (boolean) | — | Yes | *(empty)* | Boolean well-selection mask over the plate wells. One of two mutually-exclusive selection modes — either `selectionInfo`+`pattern`, or `useSectionExpression: true`+`sectionExpression`. |
| `pattern` | comArray (boolean) | — | Yes | *(derived)* | Boolean well pattern (parallel to `selectionInfo`). Always required — even when using `useSectionExpression`, the engine reads `pattern` via a throwing accessor before checking the expression flag, so include a full-length placeholder (e.g., all-false array of the correct length) when using expression-based selection. |

> **Well numbering.** Wells are 1-based and **row-major**: on a 96-well plate `A1` = 1,
> `A12` = 12, `B1` = 13, …, `H12` = 96. See [Well Numbering](../Introduction-to-Pipetting-Steps.md#well-numbering).

> **Serial Dilution selection keys vs. simple Span-8 steps.** On Serial Dilution, author `useMandrelExpression` (probe selection) and `useSectionExpression` (well selection) — not `useExpression`, which is a non-authoritative scratch key overwritten at enqueue.

### Step-Level Keys — Dilution behavior

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `transferToWaste` | boolean | — | No | `false` | Aspirate and discard the excess volume from the last well of each dilution series. |
| `prepareDiluentInPlate` | boolean | — | No | `false` | Pre-fill the plate wells with diluent from a reservoir before the serial transfers (adds diluent at `1:dilutionRatio`). |
| `prepareFirstWell` | boolean | — | No | `false` | When pre-filling, also add diluent to the first wells of the series. |

### Step-Level Keys — Main transfer, passive wash (fixed tips) / tip change

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `washBetweenTransfers` | boolean | — | Yes | `false` | Passive-wash (fixed tips) or change tips (disposable) between serial transfers. |
| `washVolume` | string (expression) | mL | Conditional | `""` | Passive-wash rinse volume (fixed tips). |
| `wasteVolume` | string (expression) | mL | Conditional | `""` | Passive-wash waste volume (fixed tips). |
| `aTipCleanMode` | string | — | Conditional | `"After Dispensing"` | **Passive** clean/change timing: `"After Dispensing"` or `"After Every Sample"`. (The `a` here is the wash-mode prefix, not "active".) |
| `preWashVolume` | string (expression) | mL | No | `"2"` | Pre-wash rinse volume applied once at the start if fixed tips are already dirty. Read by enqueue but **not written by the editor** — safe to omit. |
| `preWasteVolume` | string (expression) | mL | No | `"2"` | Pre-wash waste volume for the same start-of-run wash. Not written by the editor. |

### Step-Level Keys — Main transfer, active wash

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `activeWashBetweenTransfers` | boolean | — | Yes | `false` | Use an active wash between serial transfers. |
| `aSolvent` | string | — | Conditional | `"Water"` | Active-wash solvent liquid type. |
| `aWashVolume` | string (expression) | µL or % | Conditional | `"0"` | Active-wash volume per cycle. |
| `aWashCycles` | string (expression) | — | Conditional | `"0"` | Active-wash cycle count. |
| `aTipActiveCleanMode` | string | — | Conditional | `"After Dispensing"` | **Active** clean timing: `"After Dispensing"` or `"After Every Sample"`. |
| `activeWashTechnique` | string | — | Conditional | `""` | Active-wash technique name. |
| `autoSelectActiveWashTechnique` | boolean | — | No | `false` | Auto-select the active-wash technique. Persisted by the editor but read tolerantly (defaults to `false` if omitted); not a runtime requirement. |
| `customActiveWashTechnique` | object (technique) | — | No | *(absent)* | Inline custom active-wash technique. Present only when a custom technique is defined. |

### Step-Level Keys — Diluent transfer, passive wash / active wash

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `washBetweenDiluentTransfers` | boolean | — | No | `false` | Passive-wash (fixed tips) or change tips between diluent dispenses. Read tolerantly (defaults to `false` if omitted). |
| `diluentWashVolume` | string (expression) | mL | Conditional | `"2"` | Diluent passive-wash rinse volume. |
| `diluentWasteVolume` | string (expression) | mL | Conditional | `"2"` | Diluent passive-wash waste volume. |
| `activeWashBetweenDiluentTransfers` | boolean | — | No | `false` | Use an active wash between diluent transfers. Read tolerantly (defaults to `false` if omitted). |
| `aDilSolvent` | string | — | Conditional | `"Water"` | Diluent active-wash solvent. |
| `aDilWashVolume` | string (expression) | µL or % | Conditional | `"0"` | Diluent active-wash volume per cycle. |
| `aDilWashCycles` | string (expression) | — | Conditional | `"0"` | Diluent active-wash cycle count. |
| `aDilWashTechnique` | string | — | Conditional | `""` | Diluent active-wash technique name (root name `"ADilWashTechnique"`). |
| `autoSelectADilWashTechnique` | boolean | — | No | `false` | Auto-select the diluent active-wash technique. Read tolerantly (defaults to `false` if omitted). |
| `customADilWashTechnique` | object (technique) | — | No | *(absent)* | Inline custom diluent active-wash technique. Present only when defined. |

### Step-Level Keys — Technique sub-dictionaries (nested dictionary objects)

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `aspirateProperties` | object | Yes | Technique/height for the serial-transfer **aspirate**. See sub-dict shape below. |
| `dispenseProperties` | object | Yes | Technique/height for the serial-transfer **dispense**. |
| `diluentAspirateProperties` | object | Conditional | Source config + technique/height for the **diluent aspirate** (from a reservoir). Required when `prepareDiluentInPlate` is `true`. |
| `diluentDispenseProperties` | object | Conditional | Technique/height for the **diluent dispense** into the plate. |

### Inert / UI-state keys

The keys below are written by the editor but never read by the enqueue path — they are
UI mirrors or panel state. Safe to omit; preserve verbatim on round-trip if present.

| Key | Written by editor | Classification | Notes |
|-----|-------------------|----------------|-------|
| `amount` | always | inert UI-mirror | Mirror of `volume`. The engine reads `volume`, never `amount`. |
| `firstLoad` | always | inert UI-state | Tracks whether the "Load Tips" checkbox has been applied once; UI bookkeeping only. |
| `displayDiluentProps` | always | inert UI-state | Collapse state of the diluent panel. Cosmetic. |
| `isUserSelected` | conditional | inert UI-state | Tracks whether the tip type was manually chosen. Cosmetic. |
| `defaultCaption` | always | base | See [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md). |

## Nested sub-dictionary shapes

### `aspirateProperties` / `dispenseProperties`
Plain dictionary (no `_biomekType`). Keys written by the editor:

| Sub-key | Type | Meaning |
|---------|------|---------|
| `prototype` | string | Technique name. **Required whenever the parent sub-dictionary is present** — the key must appear (the value may be `""` when `autoSelectPrototype` is `true`), or the step fails. |
| `autoSelectPrototype` | boolean | Auto-select technique. **Required whenever the parent sub-dictionary is present** — omitting the key fails the step, regardless of its value. |
| `customPrototype` | object | Custom technique (only when defined) |
| `operation` | string | `"Aspirate"` / `"Dispense"` |
| `height` | number/expression | Tip height offset. **Required whenever the parent sub-dictionary is present** — the key must appear even when `overrideHeight` is `false`. |
| `heightFrom` | integer | `0`=Liquid, `1`=Bottom, `2`=Top. **Required whenever the parent sub-dictionary is present**, on the same terms as `height`. |
| `overrideHeight` | boolean | Override technique height. **Required whenever the parent sub-dictionary is present** — omitting the key fails the step, regardless of its value. |
| `customHeight` | boolean | Encoding discriminator for `height`: `false` = numeric offset; `true` = text/custom height, which also implies `overrideHeight`. See [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#height-encoding-keys). **Required whenever the parent sub-dictionary is present**, on the same terms as `height` and `heightFrom`. |
| `firstWell` | integer | 1-based first well |
| `firstWellExpression` | string | First-well expression |
| `useWellExpression` | boolean | When true use `firstWellExpression`, when false use the well index `firstWell`. |

See [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md)
for `prototype`/`height` semantics.

### `diluentAspirateProperties`
Everything in `aspirateProperties`, **plus** the diluent source config and a well selection:

| Sub-key | Type | Meaning |
|---------|------|---------|
| `diluentLocation` | string | Deck position of the diluent reservoir |
| `diluentLiquidType` | string | Diluent liquid type |
| `diluentLabwareClass` | string | Reservoir labware class (must be a reservoir) |
| `selectionInfo` / `pattern` | comArray (boolean) | Reservoir section selection |
| `wellsX` | integer | Reservoir columns |
| `sectionExpression` / `useExpression` | string / boolean | Expression-based section selection |

### `diluentDispenseProperties`
Same shape as `aspirateProperties`/`dispenseProperties` (technique + height + well-selection keys).

## Enumerated / Constrained Values

### direction
| Value | Meaning |
|-------|---------|
| `"Left to Right"` | Dilute along rows, left→right (probes may address multiple rows at once). |
| `"Top to Bottom"` | Dilute along columns, top→bottom (single-probe pipetting). |

Both are source constants and the only legal values (Rule 4).

### aTipCleanMode / aTipActiveCleanMode
| Value | Meaning |
|-------|---------|
| `"After Dispensing"` | Clean/change tips after every dispense. |
| `"After Every Sample"` | Clean/change tips only between samples (dilution series). |

Both are source constants.

## Cross-Field Validation Rules

Each rule describes the enqueue-time behavior it enforces.

1. **Span-8 pod required** — the pod must be a Span-8 pod; an unresolvable pod name is a distinct error.
2. **Position present** — an unresolved `where`/`position` fails validation.
3. **Configured** — missing labware/pattern fails validation; the editor summary shows a "not yet configured" placeholder when `aspirateProperties` is absent.
4. **Direction** — anything other than the two legal values is rejected.
5. **Labware kind** — the dilution plate must be a titerplate or tube rack; any other labware class is rejected.
6. **Contiguous selection** — the selected wells must be contiguous along `direction`; a gap fails validation.
7. **Probes selected** — the resolved probe mask must not be empty.
8. **Tips loaded when not loading** — with `loadTips: false` there must be tips on the pod.
9. **Reachability** — the selected probes must be able to reach every row/column; unreachable rows/columns are reported.
10. **Diluent (only when `prepareDiluentInPlate` is `true`)** — the diluent side has its own configuration checks: labware type, labware kind (reservoir), section selection, position, and ratio.

## Structural Context

This step is a **leaf** — no authored `subSteps`.

## Canonical Example

Minimal Left-to-Right dilution on a 96-well plate, disposable tips, tip change between
transfers, no diluent pre-fill. The `pattern` and `selectionInfo` comArrays below are
full-length (96 values, one per well of a 12-column x 8-row plate) — columns 1-4 are selected. Array
length must equal the plate's well count; a truncated pattern trips the "not configured" check.

> The Span-8 pod is `Pod1` on a standalone Span-8 instrument (i5-Span, i7-Span) and `Pod2`
> (the second/rightmost pod) on an i7 Hybrid. The example uses `Pod1` because it is valid on
> a standalone Span-8 — use whichever pod your instrument
> defines.

```json
{
  "stepType": "Span-8 Serial Dilution",
  "parameters": {
    "aDilSolvent": "Water",
    "aDilWashCycles": "0",
    "aDilWashTechnique": "",
    "aDilWashVolume": "0",
    "aSolvent": "Water",
    "aTipActiveCleanMode": "After Dispensing",
    "aTipCleanMode": "After Dispensing",
    "aWashCycles": "0",
    "aWashVolume": "0",
    "activeWashBetweenDiluentTransfers": false,
    "activeWashBetweenTransfers": false,
    "activeWashTechnique": "",
    "amount": "20",
    "aspirateProperties": {
      "autoSelectPrototype": true, "customHeight": false, "firstWell": 1,
      "firstWellExpression": "", "height": 0.0, "heightFrom": 0,
      "operation": "Aspirate", "overrideHeight": false, "prototype": "",
      "useWellExpression": false
    },
    "autoSelectADilWashTechnique": false,
    "autoSelectActiveWashTechnique": false,
    "diluentAspirateProperties": {
      "autoSelectPrototype": true, "customHeight": false,
      "diluentLabwareClass": "", "diluentLiquidType": "", "diluentLocation": "",
      "firstWell": 1, "firstWellExpression": "", "height": 0.0, "heightFrom": 0,
      "operation": "Aspirate", "overrideHeight": false, "prototype": "",
      "sectionExpression": "", "useExpression": false, "useWellExpression": false, "wellsX": 1
    },
    "diluentDispenseProperties": {
      "autoSelectPrototype": true, "customHeight": false, "firstWell": 1,
      "firstWellExpression": "", "height": 0.0, "heightFrom": 2,
      "operation": "Dispense", "overrideHeight": false, "prototype": "",
      "useWellExpression": false
    },
    "diluentWashVolume": "2",
    "diluentWasteVolume": "2",
    "dilutionRatio": "2",
    "direction": "Left to Right",
    "dispenseProperties": {
      "autoSelectPrototype": true, "customHeight": false, "firstWell": 1,
      "firstWellExpression": "", "height": 0.0, "heightFrom": 2,
      "operation": "Dispense", "overrideHeight": false, "prototype": "",
      "useWellExpression": false
    },
    "displayDiluentProps": false,
    "firstLoad": true,
    "labwareClass": "BCFlat96",
    "leaveTipsOn": true,
    "liquidtype": "Water",
    "loadTips": true,
    "mandrelExpression": "",
    "pattern": {
      "_biomekType": "comArray", "arraySubtype": "boolean",
      "values": [
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false
      ]
    },
    "pod": "Pod1",
    "position": "P13",
    "prepareDiluentInPlate": false,
    "prepareFirstWell": false,
    "selectionInfo": {
      "_biomekType": "comArray", "arraySubtype": "boolean",
      "values": [
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false,
        true, true, true, true, false, false, false, false, false, false, false, false
      ]
    },
    "sectionExpression": "",
    "tipLocation": "BC230",
    "tips": "BC230",
    "transferToWaste": false,
    "useCurrentTips": true,
    "useDisposableTips": true,
    "useExpression": false,
    "useFixedTips": false,
    "useMandrelExpression": false,
    "useMandrelSelection": false,
    "useProbes": {
      "_biomekType": "comArray", "arraySubtype": "boolean",
      "values": [true, true, true, true, true, true, true, true]
    },
    "useSectionExpression": false,
    "volume": "20",
    "washBetweenDiluentTransfers": false,
    "washBetweenTransfers": true,
    "washVolume": "2",
    "wasteVolume": "2",
    "wellsX": 12,
    "what": "BCFlat96",
    "where": "P13"
  }
}
```

## Common Mistakes

- **Non-contiguous well selection along `direction`**: the selected wells must form a
  contiguous run along the chosen `direction` (`"Left to Right"` or `"Top to Bottom"`).
  A gap trips the contiguity check (Rule 6). Gaps in the perpendicular axis are allowed.
- **Using a non-titerplate/tuberack**: the dilution plate must be a titerplate or tube
  rack. Any other labware class (reservoir, tip box, etc.) fails Rule 5.
- **Forgetting to mirror `where`/`position` and `what`/`labwareClass`**: both mirror-pairs
  must carry the same value. Omitting either half fails validation.
- **Confusing `aTipCleanMode` with `aTipActiveCleanMode`**: `aTipCleanMode` is the
  **passive** clean/change timing (fixed tips or disposable tip-change) and pairs with
  `washBetweenTransfers`. `aTipActiveCleanMode` is the **active** clean timing and pairs
  with `activeWashBetweenTransfers`. They are not interchangeable.
- **Omitting diluent-side properties when `prepareDiluentInPlate: true`**: turning on
  diluent pre-fill requires `diluentAspirateProperties` (with source config —
  `diluentLocation`, `diluentLabwareClass`, `diluentLiquidType`, and a reservoir well
  selection) **and** `diluentDispenseProperties`. Missing/empty either → the diluent
  validation strings in Rule 10 fire.

## Related Guides

- **[Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md)** — technique/height keys inside the four `*Properties` sub-dicts.
- **[Method JSON Structure](../Method-JSON-Structure.md)** — `comArray` encoding for `useProbes`/`selectionInfo`/`pattern`.
- **[Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)** — base keys, casing, `disabled`.
