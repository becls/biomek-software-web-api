# Combine

| Property | Value |
|----------|-------|
| stepType | `"Combine"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7 — see note below |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Universal base keys (`caption`, `defaultCaption`, `dynamic?`, …) are documented there, not here.

> **Implementation note.** Combine shares implementation with **Transfer**. Because the two steps share every key,
> **this doc mirrors [Transfer](11-Transfer.md)**; only the Combine-specific deltas are
> spelled out in full. Read the Transfer doc for the deep item-level detail and the shared
> `_biomekType: "comArray"` shapes.

---

## Behavior Summary

The Combine step is the mirror image of Transfer: it moves liquid from **multiple source
locations into a single destination** (many → one), whereas Transfer is one → many. At enqueue
time it verifies that all source wells and destination wells are defined with
proper selections, resolves both source and destination labware positions and
classes, and builds one or more transfer groups optimized for efficient
pipetting. Tip handling, washing, repeats/replicates, and large-volume splitting are
identical to Transfer.

The primary difference from Transfer is that the list of items includes multiple
sources and a single destination, and transfer volumes are defined on each
source, rather than on the destination.

> **Hardware note.** Combine is not available on a Fixed‑8 pod, unlike Transfer. The shared
> engine supports Span‑8 and multichannel pods for Combine; name the pod you want
> and set the `span8`/`podType` discriminators to match it.

---

## Parameters Reference Table

> **Note.** Combine is implemented as a subclass of the Transfer step and shares Transfer's entire enqueue implementation, so its key set, defaults, and required-ness are identical to Transfer — only the compatible hardware, caption, and `Type` value differ.

Every step-level and item-level key is **identical to Transfer**. The table below is the Transfer
key set with Combine-specific semantics called out; see [Transfer](11-Transfer.md) for the
full per-key prose, enum tables, and comArray shapes. Keys marked **(Combine delta)** differ in
meaning or presence from Transfer.

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `type` | string | — | Yes | `"Combine"` | **(Combine delta)** Must be set to `"Combine"`. |
| `pod` | string | — | Yes | `""` | Name of the pipetting pod (e.g., `"Pod1"`). Must resolve at runtime to a valid pod (**(Combine delta)** Span-8 or Multichannel only.). |
| `podType` | string | — | No | derived from `pod` | Pod-family label (`"Span8"`/`"MC"` — see §podType). **Keep it in sync with the resolved pod.** Which pod family actually runs is resolved from the named `pod`, so `podType` does not select the family; but it must still be correct, because it selects the pod-family wording of the printed method (mandrel-vs-section labels, Span-8 lines, tip count, Combine/Transfer header) — a stale value prints a method that contradicts the pod. |
| `span8` | boolean | — | No | (absent) | Span-8 pod-family discriminator. **Keep it in sync with the resolved pod.** Set to `true` for a Span-8 pod and `false` for any other pod. |
| `items` | array (object[]) | — | Yes | `[]` | Array of source and destination item dictionaries. Each item describes a labware location, well selection pattern, volume, and pipetting parameters. Position is what assigns the role: **(Combine delta)** all items **except the last item** are sources; the **last** item is the **destination**. The array must contain at least one of each (so at least two entries). The item-level `source` key is written positionally at enqueue and any authored value is overwritten (see the `source` row in Item-Level Keys). See Item-Level Keys subsection below. |
| `stop` | string | — | No | `"Sources"` | Stopping condition: `"Sources"` (exhaust all sources), `"Destinations"` (fill all destinations), or `"Either"` (stop when either is exhausted). Controls iteration over wells in Transfer assignment. The fallback default differs by pod family. Author it explicitly. |
| `repeats` | string | — | No | `"1"` | This key is used for configuring multidispense behavior. The default is to allow a single dispense for each aspirate, represented by a value of 1 for this key. If `repeatsByVolume` is `true`, this is the maximum volume in microliters that can be aspirated at once for repeated dispensing (i.e. aspirate once, dispense multiple times). If `repeatsByVolume` is `false`, this is the maximum number of dispenses allowed for a single aspirate (minimum 1). Multidispensing is not a common usage and requires specialized pipetting techniques. This should typically be authored and set to `"1"`. |
| `repeatsByVolume` | boolean | — | No | `false` | When `true`, `repeats` specifies the maximum volume to aspirate for multidispensing; when `false`, `repeats` specifies the maximum number of dispenses allowed per aspirate. |
| `replicates` | string | — | No | `"1"` | Number replicates to create for each source well. Expression-capable. Minimum 1. This key impacts the mapping from source wells to destination wells, ensuring that each source well is transferred to the given number of destination wells before moving on to the next source well. |
| `useJIT` | boolean | — | No | `true` | When `true`, aspirate and dispense operations are bundled into a single Just-In-Time block to ensure no other actions occur in between the aspirate and dispense. |
| `useFixedTips` | boolean | — | No | `false` | One of the Span-8 tip-mode keys. When `true`, only fixed-tip probes are used and `useProbes` is ignored (tip physicality is inferred from the selected probes). Do not also set `useDisposableTips`. |
| `useDisposableTips` | boolean | — | No | `false` | One of the Span-8 tip-mode keys. When `true`, all non-fixed (disposable-tip) probes are used and `useProbes` is ignored. Do not also set `useFixedTips`. |
| `useMandrelSelection` | boolean | — | No | `true` | Inert UI-state (engine does not read it; it persists the editor's selection mode). Execution branches on `mandrelExpression`/`useProbes` (UseExpression), `useFixedTips`, and `useDisposableTips`. |
| `useProbes` | comArray (boolean[8]) | — | Conditional | All `true` | Array of eight booleans indicating which probes (1–8) are active on Span-8 pods. Serialized as `{"_biomekType": "comArray", "arraySubtype": "boolean", "values": [...]}`. Required (with at least one `true` element) when `useFixedTips`, `useDisposableTips`, and `useExpression` are all `false` — i.e., SELECTION mode sourced from `useProbes`. In FIXED (`useFixedTips: true`) or DISPOSABLE (`useDisposableTips: true`) mode, or when `useExpression` is `true`, `useProbes` is ignored. Multichannel ignores this key — MC head geometry (96 or 384 channels) is not per-probe selectable at the step level. |
| `useExpression` | boolean | — | No | `false` | When `true`, probe selection is determined by `mandrelExpression` instead of `useProbes`. |
| `mandrelExpression` | string | — | Conditional | `""` | Expression that evaluates to the probe selection at runtime (e.g., `"=[1,2,3,4,5,6,7,8]"` for Span-8). Required when `useExpression` is `true`. |
| `numberOfTips` | string | — | No | `"8"` | **(Combine delta)** **Inert on Combine.** This key is not used on Combine as it only applies to Fixed-8 pods, which are not supported for Combine. Do not author it. |
| `useCurrentTips` | boolean | — | No | `false` | When `true`, the step reuses tips already loaded on the pod from a previous step instead of loading new tips. |
| `changeTipsBetweenSources` | boolean | — | No | `true` on Multichannel, `false` on Span-8 | For a Multichannel pod, when `true`, tips are changed (or washed) between aspirations from different source locations. For a Span-8 pod, whether or not tips are changed/washed is the result of a logical OR of `changeTipsBetweenSources` and `changeTipsBetweenDestinations`. When authoring a Span-8 Combine step, leave this set to `false` and utilize `changeTipsBetweenDests`. **(Combine delta)** — for many→one, sources are iterated, so this is the primary tip-change control. |
| `changeTipsBetweenDests` | boolean | — | No | `false` on Multichannel, `true` on Span-8 | When `true`, tips are changed (or washed) between after dispensing. While this key may suggest it applies when destinations are different, it actually should be interpreted as "change (or wash) tips after dispensing". |
| `leaveTipsOn` | boolean | — | No | `false` | When `true`, tips are left on the pod after the step completes; when `false`, tips are unloaded. |
| `splitVolume` | boolean | — | No | `false` | When `true`, a transfer volume larger than the tip capacity is split into several aspirate-dispense cycles. |
| `splitVolumeCleaning` | boolean | — | No | `false` | When `true` and `splitVolume` is `true`, tips are cleaned between each split cycle (if `changeTipsBetweenSources` or `changeTipsBetweenDests` is `true`). |
| `tipLocation` | string | — | Conditional | `""` | Tip source used when loading fresh tips — a tip-box labware **name**, a deck position (e.g. `"P1"`), or a tip labware **class** (e.g. `"BC230"`). Required when loading new disposable tips (i.e., when `useCurrentTips` is `false` and `useFixedTips` is `false`). **Class-name resolution only matches a box whose instance name is empty or equal to the class name** — when the tip box on deck carries a distinct `properties.name` (renamed instance), author the box's instance name or its deck position. |
| `unloadLocation` | string | — | No | `"<where they came from>"` | **(Combine delta)** **Inert on Combine.** This key is not used on Combine as it only applies to Fixed-8 pods, which are not supported for Combine. Do not author it. |
| `wash` | boolean | — | No | `false` | For Multichannel and Span-8 pods: when `true`, an active wash (using specified solvent and wash technique) is performed between tip handling events. Mutually exclusive with `span8Wash`. |
| `span8Wash` | boolean | — | No | `false` | For Span-8 pods: when `true`, a passive wash is performed (uses `span8WashVolume` and `span8WasteVolume`). Mutually exclusive with `wash`. |
| `washVolume` | string | µL or % | Conditional | `""` | Volume of solvent to use in active wash, or percentage of previously used tip capacity (e.g., `"110%"`). Expression-capable. Required when `wash` is `true`. |
| `span8WashVolume` | string | mL | No | `"2"` | Volume of passive wash solvent for Span-8 pods (e.g., `"2"`), in **mL**. Expression-capable. Default 2 mL. |
| `span8WasteVolume` | string | mL | No | `"1"` | Volume of waste liquid discarded after passive wash for Span-8 pods (e.g., `"1"`), in **mL**. Expression-capable. Default 1 mL. |
| `washCycles` | string | — | Conditional | `""` | Number of wash cycles to perform (e.g., `"3"`). Expression-capable. Required when `wash` is `true`. |
| `solvent` | string | — | Conditional | `"Water"` | Name of the washing solvent (e.g., `"Water"`, `"Ethanol"`). Required when `wash` is `true`. When configuring an active wash, the `solvent` must match the liquid type configured for an active wash station ALP on the deck. |
| `autoSelectActiveWashTechnique` | boolean | — | No | `false` | When `true`, an active wash technique is auto-selected based on liquid type and pod; when `false`, use `activeWashTechnique` (if a string) or `customActiveWashTechnique` (if an object). Tolerant read at enqueue; omission does not throw. |
| `activeWashTechnique` | string | — | No | `""` | Name of a predefined wash technique (e.g., `"Standard Wash"`). Ignored if `autoSelectActiveWashTechnique` is `true` or `customActiveWashTechnique` is bound. |
| `customActiveWashTechnique` | object | — | No | Not present | Custom wash technique definition (inline technique object). Takes precedence over `activeWashTechnique`. |
| `showTipHandlingDetails` | boolean | — | No | `false` | UI flag: when `true`, the tip handling summary is displayed in the method editor. |
| `showTransferDetails` | boolean | — | No | `false` | UI flag: when `true`, the transfer mapping summary is displayed in the method editor. |
| `wizard` | boolean | — | No | `false` | Internal UI flag. Not read at runtime. |
| `groupTransfer` | boolean | — | No | `false` | Obsolete and unused key. Do not author. |

> _The step-level tip/split/wash/UI keys above (`useFixedTips`, `useDisposableTips`, `useMandrelSelection`, `useExpression`, `splitVolume`, `splitVolumeCleaning`, `activeWashTechnique`, `showTipHandlingDetails`, `showTransferDetails`, `wizard`) are persisted by the editor but stay Optional — omission does not throw at enqueue. However, they should be authored so that a method imported into the Biomek editor displays the appropriate values in the UI._

> _Universal base keys (`caption`, `defaultCaption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

### Item-Level Keys

Item-level keys are **identical to Transfer** — see [Transfer](11-Transfer.md) for the full
table (`source`, `position`, `labwareClass`, `volume`, `pattern`, `selectionInfo`, `operation`,
`liquidType`, `prototype`, `autoSelectPrototype`, `customPrototype`, `overrideHeight`, `height`,
`heightFrom`, `customHeight`, `colsFirst`, `rowsFirst`, `startAtMark`, `startAtSelection`,
`setMark`, `wellsX`, `firstWell`, `firstWellUsesExpression`, `useExpression`, `sectionExpression`,
`dataSetPattern`, `localPattern`, `referencedPattern`, `dataSetCondition`, `dataSetConditionType`,
`spacing`, `mixCount`, `ghosted`). Two items carry **Combine-specific
semantics**:

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `source` | boolean | — | No (auto-set) | `true` or `false` (auto-marked) | Set automatically at enqueue based on item position — **(Combine delta)** for a Combine step, the last item is the destination and every previous item is a source. Any value you author here is OVERWRITTEN, so do not set it manually. |
| `volume` | string | µL | Conditional | `""` | Volume to aspirate (if source) or dispense per well (if destination). Expression-capable. **(Combine delta)** For Combine, set the volume on each source. |

---

## Enumerated / Constrained Values

### stop

| Value | Meaning |
|-------|---------|
| `"Sources"` | Stops when all source wells are exhausted; destination assignment continues until sources run out. |
| `"Destinations"` | Stops when all destination wells are filled; source assignment continues until destinations are full. |
| `"Either"` | Stops when either sources or destinations are exhausted (whichever comes first). |

### type

| Value | Meaning |
|-------|---------|
| `"Transfer"` | One source → multiple destinations (different step). Must always be `"Combine"` for this step type. |
| `"Combine"` | Multiple sources → one destination (this step). |

### podType

| Value | Meaning |
|-------|---------|
| `"Fixed8"` | Fixed-8 pod — all 8 probes (or a subset of 1–8 via `numberOfTips`), using disposable tips. |
| `"Span8"` | Span-8 pod with 8 independent probe channels. |
| `"MC"` | Multichannel pod (96- or 384-channel head). This is the string the editor emits; `"Multichannel"` behaves identically. |

### operation (per item)

| Value | Meaning |
|-------|---------|
| `"Aspirate"` | Item is a source; liquid is aspirated from this location. |
| `"Dispense"` | Item is a destination; liquid is dispensed to this location. For Combine steps, only the last item should be `"Dispense"`. |

### liquidType (per item)

| Value | Meaning |
|-------|---------|
| `"Well Contents"` | Auto-detect liquid type from the labware's liquid data set at runtime. Valid for source (Aspirate) items only; invalid for Dispense. |
| `"Tip Contents"` | For Dispense items: use the actual liquid detected in the tips. For Aspirate: invalid. |
| Any named liquid (e.g., `"Water"`, `"Serum"`) | Specific liquid type for technique selection. Overrides auto-detection. |

### heightFrom (per item)

| Value | Meaning |
|-------|---------|
| `0` or `"Liquid"` | Measure height from the liquid surface in the well. |
| `1` or `"Bottom"` | Measure height from the bottom of the well. |
| `2` or `"Top"` | Measure height from the top of the well. |

### wash

| Value | Meaning |
|-------|---------|
| `true` | Perform active wash (Span-8 or Multichannel) using specified solvent and technique. |
| `false` | No active wash. |

### span8Wash

| Value | Meaning |
|-------|---------|
| `true` | Perform passive wash (Span-8 only) using configured wash volume and waste volume. |
| `false` | No passive wash. |

### useExpression (step-level) vs. useExpression (item-level)

**Step-level (probe-selection subvariant; meaningful only in SELECTION mode — `useFixedTips` and `useDisposableTips` both `false`)**:

- `true`: Use `mandrelExpression` to determine probe selection at runtime.
- `false`: Use `useProbes` array for static probe selection.

**Item-level**:

- `true`: Use `sectionExpression` to determine well selection at runtime.
- `false`: Use the static boolean mask — `selectionInfo` on Multichannel, `pattern` on Span-8 (see Rule 12).

---

## Cross-Field Validation Rules

Validation rules are largely the same as the Transfer step. The numbering here
matches the numbering in the Transfer step, with modifications noted with
**Combine delta**.

1. **At least one source and one destination required**: the step validates that at least one item is marked as source (`source: true`) and at least one is marked as destination (`source: false`). If either is missing, enqueue fails with a "no source/destination defined" error.

2. **Transfer type constraint**: **Combine delta** For a Combine step, only the last item is marked as the destination; all previous items are sources. Multi-destination pipetting is done with the **Transfer** step, not Combine — do not place multiple destination items in a Combine step's `items` array.

3. **Tip selection is mutually exclusive**: `useFixedTips` and `useDisposableTips` are opposite tip-handling modes on Span-8 — do **not** set both `true`. On a Span-8 disposable-tip step, leave both `false` and pick channels via `useProbes` (or `mandrelExpression` when `useExpression` is `true`); disposable tips are the Span-8 default. Setting both is non-fatal — the engine reads them tolerantly and the first mode checked wins — but it is never correct to author. When both are `false`, the step falls through to per-probe selection via `useProbes`/`mandrelExpression`.

4. **Probe array constraints**: If `useFixedTips`, `useDisposableTips`, and `useExpression` are **all `false`** (SELECTION mode sourced from `useProbes`), then `useProbes` must be present with **at least one `true` element**. If all probes are `false`, enqueue fails with a "no usable probes" error. In FIXED or DISPOSABLE mode, or when `useExpression` is `true`, `useProbes` is ignored. (`useMandrelSelection` is inert UI-state and does not participate in this check.)

5. **Expression-based probe selection**: If `useExpression` is `true`, then `mandrelExpression` **must be non-empty** and must evaluate to a valid probe array at runtime.

6. **Tip location required for fresh tips**: If `useCurrentTips` is `false` and
   `useFixedTips` is `false`, then `tipLocation` **must be non-empty** and
   resolvable to a tip box on the deck. If not resolved, enqueue fails with a
   tips-not-found error.

7. **Combine delta**: Item 7 in the Transfer step's validation rules does not
   apply, as Fixed-8 pods are not valid for Combine.

8. **All selected tips must be same type and syringe size (Span-8)**: At runtime, the step validates that all selected probes with tips have:
   - Same tip type (e.g., all Fixed or all Disposable)
   - Same syringe size (e.g., all 200 µL)
   If mismatched, enqueue fails with a same-tip-type / same-syringe-size error.

9. **Wash solvent required if wash is enabled**: If `wash` is `true`, then `solvent` **must be non-empty** and `washVolume` **must be non-empty**. The named `solvent` must resolve to a wash station on the deck; if none is found, tip cleaning throws a "no wash station" error, surfaced during the transfer as a tip-cleaning failure at the source position.

10. **Active wash vs. passive wash (Span-8)**: `wash` (active) and `span8Wash` (passive) **should not both be `true`** at the same time.

11. **Wash technique specification**: If `wash` is true and `autoSelectActiveWashTechnique` is `false`, either `activeWashTechnique` (string name) or `customActiveWashTechnique` (object) **must be non-empty or defined**. If neither is present and `autoSelectActiveWashTechnique` is `false`, the step may fail or skip washing.

12. **Well selection in each item**: every item selects wells through exactly one mode, and **which key is the operative selector depends on the pod family**:

 | Mode (item keys) | Operative selector |
 |---|---|
 | Default — `localPattern: true`, `dataSetPattern: false`, `useExpression: false`, on **Multichannel** | `selectionInfo` |
 | Default — `localPattern: true`, `dataSetPattern: false`, `useExpression: false`, on **Span-8** | `pattern` |
 | `localPattern: false` | `referencedPattern` (name of a globally-defined pattern) |
 | `dataSetPattern: true` | the named data set + `dataSetCondition`/`dataSetConditionType` |
 | `useExpression: true` | `sectionExpression` |

13. **Volume specification**: For each item, **at least one way to specify volume must be present**:
    - `volume` key (string, µL) — preferred for most items, or
    - `amount` and `amounts` — if per-probe volumes are needed (Span-8 specific)
    An **omitted** `volume` key defaults to `0` (which may cause technique validation errors). An **empty string** `""` is **not** the same as omitted — it fails at enqueue with `Invalid volume specified for <source|destination> N.` Every item that carries volume must give a non-empty numeric value or an evaluable expression; to leave it unset, omit the key rather than authoring `""`.

14. **Volume per source vs. per destination**: For a Combine step, always set the volume on the source.

15. **Repeats and repeatsByVolume constraints**:
    - If `repeatsByVolume` is `false`: `repeats` must evaluate to a number ≥ 1 indicating a number of dispenses per aspirate.
    - If `repeatsByVolume` is `true`: `repeats` must evaluate to a number indicating a volume to aspirate for multidispensing.

16. **Stop condition and well assignment**: The choice of `stop` ("Sources", "Destinations", "Either") determines how the algorithm pairs sources with destinations:
    - `"Sources"`: All source wells are processed; destination wells accessed as needed (looping through destinations when they are exhausted).
    - `"Destinations"`: All destination wells are processed; source wells accessed as needed (looping through sources when they are exhausted).
    - `"Either"`: Algorithm stops when first limit is hit.
    An invalid value raises a "stop condition" error.

17. **Split volume constraints**: If `splitVolume` is `true`:
    - **Splitting engages only when each aspirate matches a single dispense.** It is not valid when `repeats` is greater than 1 or `repeatsByVolume` is true.
    - Each split cycle inherits its technique's overhead (trailing air gap / blowout).
    - The split calculator sums those per-cycle aspirates against the source well's tracked volume, so split sources must be trackable — author `volumeType: "Known"` with realistic `evalAmounts` in Instrument Setup (see [Instrument Setup](03-Instrument-Setup.md)). An `"Unknown"` source under `splitVolume: true` cannot be validated and fails enqueue.
    - If `splitVolumeCleaning` is also `true` and either `changeTipsBetweenSources` or `changeTipsBetweenDests` is `true`, tips are cleaned (changed or washed) between split cycles.

18. **Leave tips vs. discard**: If `leaveTipsOn` is `true`, tips are left on the pod; if `false`, they are discarded to the location defined in Instrument Setup.

19. **Item count constraint**: `items` **must contain at least 2 elements** (one source, one destination).

20. **Item `labwareClass` must match (or be compatible with) the labware at `position`**: At enqueue the resolved labware class is checked against each item's `labwareClass`. A mismatch that is not a compatible substitute raises a class-mismatch error; an entirely unknown expected class raises a "labware class not found" error.

21. **`useCurrentTips` requires tips already on the pod**: If `useCurrentTips` is `true` but no tips are loaded on the pod when the step runs, enqueue fails with a "no tips loaded" error.

22. **Non-LLS tips are incompatible with LLS-enabled techniques (Span-8)**: The stock Span-8 pipetting techniques enable liquid level sensing on the initial descent, but non-LLS tip classes (e.g., `T190F`, `T50F` — filtered tips with `LLS: false` in the catalog) cannot liquid-level-sense. Combining them with the default auto-selected Span-8 technique hard-fails enqueue with a wrong-tip error. To use non-LLS tips, either pick an LLS-capable tip class or set `overrideHeight: true` on the item with a fixed-reference `heightFrom` (`1`/`"Bottom"` or `2`/`"Top"`) so the technique's initial LLS branch is bypassed.

23. **Volume tracking (cross-reference)**: The engine tracks a running per-well volume for the source and destination wells and rejects both over-dispense (fills exceeding the well/labware capacity) and under-aspirate (drawing more than the tracked source volume). Author source starting volumes and reservoir dead volumes in [Instrument Setup](03-Instrument-Setup.md) — see the "Volume tracking" note there.

24. **Combine delta**: Item 24 in the Transfer step's validation rules does not
    apply, as Fixed-8 pods are not valid for Combine.

---

## Structural Context

Leaf step — no `subSteps`.

---

## Canonical Example

Two sources combined into one destination (multichannel-style pod; `span8: false`). Well-selection
comArrays truncated for readability — supply a full boolean array per item as in
[Transfer](11-Transfer.md).

```json
{
  "stepType": "Combine",
  "parameters": {
    "type": "Combine",
    "pod": "Pod1",
    "span8": false,
    "dynamic?": true,
    "items": [
      {
        "source": true,
        "operation": "Aspirate",
        "position": "P5",
        "labwareClass": "BCFlat96",
        "liquidType": "Well Contents",
        "volume": "50",
        "prototype": "MC",
        "autoSelectPrototype": false,
        "overrideHeight": false,
        "height": 0.0,
        "heightFrom": 0,
        "customHeight": false,
        "localPattern": true,
        "startAtSelection": true,
        "startAtMark": false,
        "setMark": true,
        "colsFirst": true,
        "rowsFirst": false,
        "useExpression": false,
        "sectionExpression": "",
        "referencedPattern": "",
        "dataSetPattern": false,
        "wellsX": 12,
        "pattern": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true, false, false, false] },
        "selectionInfo": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true, false, false, false] }
      },
      {
        "source": true,
        "operation": "Aspirate",
        "position": "P6",
        "labwareClass": "BCFlat96",
        "liquidType": "Well Contents",
        "volume": "50",
        "prototype": "MC",
        "autoSelectPrototype": false,
        "overrideHeight": false,
        "height": 0.0,
        "heightFrom": 0,
        "customHeight": false,
        "localPattern": true,
        "startAtSelection": true,
        "startAtMark": false,
        "setMark": true,
        "colsFirst": true,
        "rowsFirst": false,
        "useExpression": false,
        "sectionExpression": "",
        "referencedPattern": "",
        "dataSetPattern": false,
        "wellsX": 12,
        "pattern": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true, false, false, false] },
        "selectionInfo": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true, false, false, false] }
      },
      {
        "source": false,
        "operation": "Dispense",
        "position": "P9",
        "labwareClass": "BCFlat96",
        "liquidType": "Tip Contents",
        "volume": "100",
        "prototype": "MC",
        "autoSelectPrototype": false,
        "overrideHeight": false,
        "height": 0.0,
        "heightFrom": 0,
        "customHeight": false,
        "localPattern": true,
        "startAtSelection": true,
        "startAtMark": false,
        "setMark": true,
        "colsFirst": true,
        "rowsFirst": false,
        "useExpression": false,
        "sectionExpression": "",
        "referencedPattern": "",
        "dataSetPattern": false,
        "wellsX": 12,
        "pattern": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true, false, false, false] },
        "selectionInfo": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true, false, false, false] }
      }
    ],
    "stop": "Sources",
    "repeats": "1",
    "repeatsByVolume": false,
    "replicates": "1",
    "useJIT": true,
    "useFixedTips": false,
    "useDisposableTips": false,
    "useMandrelSelection": true,
    "useProbes": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true, true, true, true, true, true, true, true] },
    "useExpression": false,
    "mandrelExpression": "",
    "useCurrentTips": false,
    "changeTipsBetweenSources": false,
    "changeTipsBetweenDests": true,
    "leaveTipsOn": false,
    "splitVolume": false,
    "splitVolumeCleaning": false,
    "tipLocation": "BC230",
    "wash": false,
    "span8Wash": false,
    "washVolume": "",
    "washCycles": "",
    "solvent": "Water",
    "span8WashVolume": "2",
    "span8WasteVolume": "1",
    "autoSelectActiveWashTechnique": false,
    "activeWashTechnique": "",
    "showTipHandlingDetails": false,
    "showTransferDetails": false,
    "wizard": false
  }
}
```

---

## Common Mistakes

- **Setting `type` to anything but `"Combine"`**: a Transfer-style value will silently reshape the semantics.
- **`items` array with fewer than 2 entries**: Combine marks items 0..N-2 as sources and the last as destination (Rule 1), so at least 2 are required. A truly single-entry `items` array fails enqueue with a "no source/destination defined" error. A 2-entry `items` (one source + one destination) is *legal* Combine and behaves like a Transfer — consider using [Transfer](11-Transfer.md) instead for clarity.
- **Omitting `useProbes` / `span8` on Span-8 hardware**: Include both when targeting a Span-8 pod.
- **Source labware-class mismatch**: When the deck position holds a class different from the source item's `items[i].labwareClass`, enqueue fails with a class-mismatch error.
