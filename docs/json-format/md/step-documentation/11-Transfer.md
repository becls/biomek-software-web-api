# Transfer

| Property | Value |
|----------|-------|
| stepType | `"Transfer"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7, i3 |

---

## Behavior Summary

The Transfer step moves liquid from a single source location to one or more destination locations in a single consolidated operation. At enqueue time, it verifies that all source wells and destination wells are defined with proper selections, resolves both source and destination labware positions and classes, and builds one or more transfer groups optimized for efficient pipetting. The step combines tip loading, aspiration from the source(s), dispense to the destination(s), and tip unload/disposal into a single logical unit.

The `stop` condition (`"Sources"`, `"Destinations"`, or `"Either"`) determines how many sources must be exhausted versus destinations filled before the step completes. The step supports multiple repetitions (`repeats`, `repeatsByVolume`), replicates (multiple times through the same sources and destinations), and large-volume splitting (when volume exceeds tip capacity, the step automatically breaks it into multiple aspirate-dispense cycles).

Tip handling is configurable: tips can be loaded fresh, reused from a previous step (`useCurrentTips`), changed between source groups, changed between destination groups, or washed between operations.

Note that the Transfer step and the Combine step share logic and keys. This
document describes the necessary keys and their values for a Transfer step (one
source, multiple destinations). To author a Combine step (multiple sources,
one destination), see [Combine](12-Combine.md).

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `type` | string | — | No | `"Transfer"` | **Do not set this key on a Transfer step.** Optional; if present, must be `"Transfer"`. |
| `pod` | string | — | Yes | `""` | Name of the pipetting pod (e.g., `"Pod1"`). Must resolve at runtime to a valid pod (Fixed-8, Span-8, or Multichannel). |
| `podType` | string | — | No | derived from `pod` | Pod-family label (`"Fixed8"`/`"Span8"`/`"MC"` — see §podType). **Keep it in sync with the resolved pod.** Which pod family actually runs is resolved from the named `pod`, so `podType` does not select the family; but it must still be correct, because it selects the pod-family wording of the printed method (mandrel-vs-section labels, Span-8 lines, tip count, Combine/Transfer header) — a stale value prints a method that contradicts the pod. |
| `span8` | boolean | — | No | (absent) | Span-8 pod-family discriminator. **Keep it in sync with the resolved pod.** Set to `true` for a Span-8 pod and `false` for any other pod. |
| `items` | array (object[]) | — | Yes | `[]` | Array of source and destination item dictionaries. Each item describes a labware location, well selection pattern, volume, and pipetting parameters. Position is what assigns the role: on a Transfer, the **first** item is the source and every subsequent item is a destination — the array must contain at least one of each (so at least two entries). The item-level `source` key is written positionally at enqueue and any authored value is overwritten (see the `source` row in Item-Level Keys). See Item-Level Keys subsection below. |
| `stop` | string | — | No | `"Destinations"` (Fixed-8) / `"Sources"` (Span-8/Multichannel) | Stopping condition: `"Sources"` (exhaust all sources), `"Destinations"` (fill all destinations), or `"Either"` (stop when either is exhausted). Controls iteration over wells in Transfer assignment. The fallback default differs by pod family. Author it explicitly. |
| `repeats` | string | — | No | `"1"` | This key is used for configuring multidispense behavior. The default is to allow a single dispense for each aspirate, represented by a value of 1 for this key. If `repeatsByVolume` is `true`, this is the maximum volume in microliters that can be aspirated at once for repeated dispensing (i.e. aspirate once, dispense multiple times). If `repeatsByVolume` is `false`, this is the maximum number of dispenses allowed for a single aspirate (minimum 1). Multidispensing is not a common usage and requires specialized pipetting techniques. This should typically be authored and set to `"1"`. |
| `repeatsByVolume` | boolean | — | No | `false` | When `true`, `repeats` specifies the maximum volume to aspirate for multidispensing; when `false`, `repeats` specifies the maximum number of dispenses allowed per aspirate. |
| `replicates` | string | — | No | `"1"` | Number replicates to create for each source well. Expression-capable. Minimum 1. This key impacts the mapping from source wells to destination wells, ensuring that each source well is transferred to the given number of destination wells before moving on to the next source well. |
| `useJIT` | boolean | — | No | `true` | When `true`, aspirate and dispense operations are bundled into a single Just-In-Time block to ensure no other actions occur in between the aspirate and dispense. |
| `useFixedTips` | boolean | — | No | `false` | One of the Span-8 tip-mode keys. When `true`, only fixed-tip probes are used and `useProbes` is ignored (tip physicality is inferred from the selected probes). Do not also set `useDisposableTips`. Fixed-8 ignores this key. |
| `useDisposableTips` | boolean | — | No | `false` | One of the Span-8 tip-mode keys. When `true`, all non-fixed (disposable-tip) probes are used and `useProbes` is ignored. Do not also set `useFixedTips`. Fixed-8 ignores this key. |
| `useMandrelSelection` | boolean | — | No | `true` | Inert UI-state (engine does not read it; it persists the editor's selection mode). Execution branches on `mandrelExpression`/`useProbes` (UseExpression), `useFixedTips`, and `useDisposableTips`. |
| `useProbes` | comArray (boolean[8]) | — | Conditional | All `true` | Array of eight booleans indicating which probes (1–8) are active on Span-8 pods. Serialized as `{"_biomekType": "comArray", "arraySubtype": "boolean", "values": [...]}`. Required (with at least one `true` element) when `useFixedTips`, `useDisposableTips`, and `useExpression` are all `false` — i.e., SELECTION mode sourced from `useProbes`. In FIXED (`useFixedTips: true`) or DISPOSABLE (`useDisposableTips: true`) mode, or when `useExpression` is `true`, `useProbes` is ignored. Fixed-8 ignores this key entirely — Fixed-8 tip count is controlled by `numberOfTips`. Multichannel also ignores this key — MC head geometry (96 or 384 channels) is not per-probe selectable at the step level. |
| `useExpression` | boolean | — | No | `false` | When `true`, probe selection is determined by `mandrelExpression` instead of `useProbes`. |
| `mandrelExpression` | string | — | Conditional | `""` | Expression that evaluates to the probe selection at runtime (e.g., `"=[1,2,3,4,5,6,7,8]"` for Span-8). Required when `useExpression` is `true`. |
| `numberOfTips` | string | — | No | `"8"` | For Fixed-8 pods: the number of tips to use (1–8 as a string, or an expression). Expression-capable. Not used on Span-8 or Multichannel pods, although the value if present must be between 1 and 8. |
| `useCurrentTips` | boolean | — | No | `false` | When `true`, the step reuses tips already loaded on the pod from a previous step instead of loading new tips. |
| `changeTipsBetweenSources` | boolean | — | No | `true` on Fixed-8/Multichannel, `false` on Span-8 | For a Multichannel or Fixed-8 pod, when `true`, tips are changed (or washed) between aspirations from different source locations. For a Span-8 pod, whether or not tips are changed/washed is the result of a logical OR of `changeTipsBetweenSources` and `changeTipsBetweenDestinations`. When authoring a Span-8 Transfer step, leave this set to `false` and utilize `changeTipsBetweenDests`. |
| `changeTipsBetweenDests` | boolean | — | No | `false` on Fixed-8/Multichannel, `true` on Span-8 | When `true`, tips are changed (or washed) between after dispensing. While this key may suggest it applies when destinations are different, it actually should be interpreted as "change (or wash) tips after dispensing". |
| `leaveTipsOn` | boolean | — | No | `false` | When `true`, tips are left on the pod after the step completes; when `false`, tips are unloaded. |
| `splitVolume` | boolean | — | No | `false` | When `true`, a transfer volume larger than the tip capacity is split into several aspirate-dispense cycles. |
| `splitVolumeCleaning` | boolean | — | No | `false` | When `true` and `splitVolume` is `true`, tips are cleaned between each split cycle (if `changeTipsBetweenSources` or `changeTipsBetweenDests` is `true`). |
| `tipLocation` | string | — | Conditional | `""` | Tip source used when loading fresh tips — a tip-box labware **name**, a deck position (e.g. `"P1"`), or a tip labware **class** (e.g. `"BC230"`). Required when loading new disposable tips (i.e., when `useCurrentTips` is `false` and `useFixedTips` is `false`). **Class-name resolution only matches a box whose instance name is empty or equal to the class name** — when the tip box on deck carries a distinct `properties.name` (renamed instance), author the box's instance name or its deck position. |
| `unloadLocation` | string | — | No | `"<where they came from>"` | Where to send tips after the transfer (**Fixed-8 only**; ignored when `leaveTipsOn` is `true`). `"<where they came from>"` (default) returns tips to their originating box; `"<Any Trash>"` discards them to a deck position carrying the "Can Discard Tips" characteristic; or give an explicit deck-position / labware name. This is the tip-*unload* vocabulary, distinct from a tip box's `DiscardTipsLocation` routing key — `"<TipBox>"` is **not** valid here. For Multichannel and Span-8 pods, this key is not used, and tips are unloaded to the location configured in the Instrument Setup step. |
| `wash` | boolean | — | No | `false` | For Multichannel and Span-8 pods: when `true`, an active wash (using specified solvent and wash technique) is performed between tip handling events. Mutually exclusive with `span8Wash`. Fixed-8 pods do not support washing; do not use this key for Fixed-8 pods. |
| `span8Wash` | boolean | — | No | `false` | For Span-8 pods: when `true`, a passive wash is performed (uses `span8WashVolume` and `span8WasteVolume`). Mutually exclusive with `wash`. |
| `washVolume` | string | µL or % | Conditional | `""` | Volume of solvent to use in active wash, or percentage of the previously used tip capacity (e.g., `"110%"`). Expression-capable. Required when `wash` is `true`. |
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

Each item in the `items` array is an object with the following keys:

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `source` | boolean | — | No (auto-set) | `true` or `false` (auto-marked) | Set automatically at enqueue based on item position — for a Transfer step, item 0 is the source and every subsequent non-ghosted item is a destination (for Combine, all items but the last are sources). Any value you author here is OVERWRITTEN, so do not set it manually. (Multi-source is a Combine step, not Transfer.) |
| `position` | string | — | Yes | `""` | Deck position name (e.g., `"P9"`) or labware name. Resolves to the labware to aspirate from (if source) or dispense to (if destination). |
| `labwareClass` | string | — | Yes | `""` | Labware class name (e.g., `"BCFlat96"`, `"CostarFlat384Square"`). Must match the class of labware at `position`. |
| `volume` | string | µL | Conditional | `""` | Volume to aspirate (if source) or dispense per well (if destination). Expression-capable. For Transfer, set the volume on each destination. |
| `pattern` | comArray (boolean[...]) | — | Conditional | Empty | Boolean array of well selections (one element per well in labware). Serialized as `{"_biomekType": "comArray", "arraySubtype": "boolean", "values": [...]}`. Each `true` indicates a well to be included. One of three mutually-usable well-selection modes: `pattern`, `dataSetPattern: true` + condition, or `useExpression: true` + `sectionExpression` (see Rule 12). **Requirement varies by pod family:** on **Span-8** `pattern` is the operative well selector whenever `localPattern` is `true` (the default) and is **hard-required** there — it must be present and correct even when `useExpression: true`, so supply a full-length placeholder (e.g. an all-false array of the labware's well count) in expression mode. On **Multichannel** the key must be present no matter which selection mode you use, and its value is superseded by the resolved section selection. On **Fixed-8** the operative selector is `selectionInfo` instead (see the next row); `pattern` is not consulted. A whole-plate operation is expressed as an all-`true` values array of the labware's well count (e.g., 96 `true`s for a 96-well plate — pair with the parallel `selectionInfo` on Fixed-8/Multichannel). |
| `selectionInfo` | comArray (subtype varies by pod) | — | Conditional | Same as pattern | Item-level well selection, kept identical to `pattern` by the editor. **Must be present and correct on Fixed-8 and Multichannel:** on the default boolean-mask path (`localPattern: true`, `useExpression: false`, `dataSetPattern: false`) `selectionInfo` **is** the operative well selector for those pod families, and a missing/invalid value fails the step with `Bad selection array.`. On Span-8 the operative selector is `pattern` (previous row), but keep `selectionInfo` in sync there too — it selects the well-selection wording of the printed method. Do not read "per-item" as implying per-well addressing on that pod; see [§Multichannel: what a Transfer item can and cannot address](#multichannel-what-a-transfer-item-can-and-cannot-address). |
| `operation` | string | — | Conditional | `"Aspirate"` (source) / `"Dispense"` (dest) | Pipetting role of the item — `"Aspirate"` for source items, `"Dispense"` for destinations. **Read at enqueue when the transfer splits a volume across multiple aspirates** (`splitVolume: true`): the authored item's value drives technique and split-count selection, and omitting it there hard-fails enqueue. On a plain non-splitting transfer the role is assigned positionally and a literal operation is used, so the key is not consulted there. The editor always emits it — **author it to match each item's role** so a splitting method still enqueues. |
| `liquidType` | string | — | Yes | `"Well Contents"` (source) or `"Tip Contents"` (dest) | Liquid type for technique selection: `"Well Contents"` (aspiration will auto-detect liquid from the deck), `"Tip Contents"` (for dispense, uses actual tip contents), or a named liquid type (e.g., `"Serum"`). Required: omitting it raises at enqueue. Real exports and the included example emit this item-level key lowercase (`liquidtype`); matching is case-insensitive, so either spelling imports correctly. Authoring tools should not depend on the exact case. |
| `prototype` | string | — | Yes (key must be present; may be `""`) | `""` | Name of the pipetting technique (transfer prototype). The **key must be present** on every item — the enqueue path reads it through an unconditional throwing accessor, so omitting it hard-fails enqueue **even when `autoSelectPrototype` is `true`**. On Fixed-8 it must name a real technique (auto-select is invalid there — see `autoSelectPrototype`); on Span-8/Multichannel with `autoSelectPrototype: true` the value may be empty (`""`), but the key still has to appear. |
| `autoSelectPrototype` | boolean | — | No | `false` | When `true`, the technique is auto-selected based on liquid type, labware, tip type, and volume — valid **only for Span-8/Multichannel pods**. **INVALID for Fixed-8.** When `false`, `prototype` must be specified. |
| `customPrototype` | object | — | No | Not present | Custom technique definition (inline technique object). Takes precedence over `prototype` when present. |
| `overrideHeight` | boolean | — | Yes | `false` | When `true`, tip height is overridden using `height` and `heightFrom` instead of the technique's defaults. Required: omitting it raises at enqueue. |
| `height` | number or expression-capable string | mm | Yes (key must be present) | `0.0` | Tip height offset from reference point when `overrideHeight` is `true`. Positive values move the tip above the reference point; negative values move it below. For `heightFrom = "Liquid"`, a value of `-2.0` places the tip **2 mm below** the liquid surface (i.e., into the liquid) — this is the standard aspirate offset used by the default techniques. Expression-capable (prefix with `=`). The **key must be present** on every item — the enqueue path reads it through an unconditional throwing accessor — so omitting it hard-fails enqueue even when `overrideHeight` is `false`; the numeric value only takes effect when `overrideHeight` is `true`. |
| `heightFrom` | integer or string | — | Conditional | `0` | Reference point for height: `0` or `"Liquid"` = liquid surface, `1` or `"Bottom"` = well bottom, `2` or `"Top"` = well top. String aliases are case-insensitive. Also supports expressions. |
| `customHeight` | boolean | — | No | `false` | Encoding discriminator for `height`: `false` = `height` is a numeric offset; `true` = `height` is a text/custom height and `overrideHeight` is treated as `true`. Use `false` with numeric heights. See [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#height-encoding-keys). |
| `colsFirst` | boolean | — | No | `false` | When `true`, well selection iteration proceeds column-first (top-to-bottom within a column, then right to the next column); when `false`, row-first. |
| `rowsFirst` | boolean | — | No | `false` | When `true`, well selection iteration proceeds row-first (left-to-right within a row, then down to the next row). If both `colsFirst` and `rowsFirst` are `false`, defaults to column-first. |
| `startAtMark` | boolean | — | No | `false` | **Span-8 Only.** When `true`, well iteration starts at the mark position on the labware (requires the deck to have marked this labware). When `false`, starts at `startAtSelection` position. |
| `startAtSelection` | boolean | — | No | `true` | **Span-8 Only.** When `true`, iteration starts at the first selected well; when `false`, at the mark or position 1. |
| `setMark` | boolean | — | No | `true` | **Span-8 Only.** When `true`, the mark on the labware is updated after the transfer completes to the last well accessed. |
| `wellsX` | integer | — | No | Auto-detected | Number of columns in the labware (e.g., `12` for a 96-well plate). Auto-detected from labware class if not provided; used for pattern indexing. |
| `firstWell` | integer | — | No | `1` | 1-based index of the first well to access if using well-number-based selection (not pattern-based). |
| `firstWellUsesExpression` | boolean | — | No | `false` | When `true`, `firstWell` is interpreted as an expression (e.g., `"=row"`). |
| `useExpression` | boolean | — | No | `false` | When `true`, well selection is determined by `sectionExpression` instead of `pattern`. |
| `sectionExpression` | string | — | Conditional | `""` | Expression that evaluates to a well index or section when `useExpression` is `true`. |
| `dataSetPattern` | boolean | — | No | `false` | **Span-8 Only.** When `true`, well selection is derived from a data set using `dataSetCondition` and `dataSetConditionType`. |
| `localPattern` | boolean | — | No | `true` | **Span-8 Only.** When `true`, `pattern` is internal to the step; when `false`, `referencedPattern` is used. |
| `referencedPattern` | string | — | Conditional | `""` | **Span-8 Only.** Name of a globally-defined pattern (from Well Patterns). Required when `localPattern` is `false`. |
| `dataSetCondition` | string | — | No | `""` | **Span-8 Only.** Condition string for data set filtering (e.g., `">0"`). Used when `dataSetPattern` is `true`. |
| `dataSetConditionType` | string | — | No | `""` | **Span-8 Only.** Type of condition: e.g., `"Greater"`, `"Less"`, `"Equal"`. Used with `dataSetCondition`. |
| `spacing` | integer | — | No | Auto-detected | Well spacing between selected probes in a Multichannel or Fixed-8 context (e.g., `1` for 96-well, `2` for 384-well). Auto-detected from labware minimum interval if not set. |
| `mixCount` | integer | — | No | `0` | Unused in Transfer steps. Do not author this key. |
| `ghosted` | boolean | — | No | `false` | When `true`, this item is ignored during transfer. Used internally for deferred items. Do not author this key; if a method is opened in the editor, these items will be removed. |

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
| `"Transfer"` | One source → multiple destinations (this step). |
| `"Combine"` | Multiple sources → one destination (different step). Must always be `"Transfer"` for this step type. |

### podType

| Value | Meaning |
|-------|---------|
| `"Fixed8"` | Fixed-8 pod — all 8 probes (or a subset of 1–8 via `numberOfTips`), using disposable tips. |
| `"Span8"` | Span-8 pod with 8 independent probe channels. |
| `"MC"` | Multichannel pod (96- or 384-channel head). This is the string the editor emits; `"Multichannel"` behaves identically. |

### operation (per item)

| Value | Meaning |
|-------|---------|
| `"Aspirate"` | Item is a source; liquid is aspirated from this location. For Transfer steps, only the first item should be `"Aspirate"`. |
| `"Dispense"` | Item is a destination; liquid is dispensed to this location. |

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
- `false`: Use the static boolean mask — `selectionInfo` on Fixed-8/Multichannel, `pattern` on Span-8 (see Rule 12).

---

## Cross-Field Validation Rules

1. **At least one source and one destination required**: the step validates that at least one item is marked as source (`source: true`) and at least one is marked as destination (`source: false`). If either is missing, enqueue fails with a "no source/destination defined" error.

2. **Transfer type constraint**: For a Transfer step, only the first item (index 0) is marked as the source; all subsequent items are destinations. Multi-source pipetting is done with the **Combine** step (`type: "Combine"`), not Transfer — do not place multiple source items in a Transfer step's `items` array.

3. **Tip selection is mutually exclusive**: `useFixedTips` and `useDisposableTips` are opposite tip-handling modes on Span-8 — do **not** set both `true`. On a Span-8 disposable-tip step, leave both `false` and pick channels via `useProbes` (or `mandrelExpression` when `useExpression` is `true`); disposable tips are the Span-8 default. Setting both is non-fatal — the engine reads them tolerantly and the first mode checked wins — but it is never correct to author. When both are `false`, the step falls through to per-probe selection via `useProbes`/`mandrelExpression`.

4. **Probe array constraints**: If `useFixedTips`, `useDisposableTips`, and `useExpression` are **all `false`** (SELECTION mode sourced from `useProbes`), then `useProbes` must be present with **at least one `true` element**. If all probes are `false`, enqueue fails with a "no usable probes" error. In FIXED or DISPOSABLE mode, or when `useExpression` is `true`, `useProbes` is ignored. (`useMandrelSelection` is inert UI-state and does not participate in this check.)

5. **Expression-based probe selection**: If `useExpression` is `true`, then `mandrelExpression` **must be non-empty** and must evaluate to a valid probe array at runtime.

6. **Tip location required for fresh tips**: If `useCurrentTips` is `false` and `useFixedTips` is `false`, then `tipLocation` **must be non-empty** and resolvable to a tip box on the deck. If not resolved, enqueue fails with a tips-not-found error.

7. **Number of tips constraint**: `numberOfTips` must be a string evaluating to an integer between 1 and 8. If invalid, tip selection fails.

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
 | Default — `localPattern: true`, `dataSetPattern: false`, `useExpression: false`, on **Fixed-8 / Multichannel** | `selectionInfo` |
 | Default — `localPattern: true`, `dataSetPattern: false`, `useExpression: false`, on **Span-8** | `pattern` |
 | `localPattern: false` | `referencedPattern` (name of a globally-defined pattern) |
 | `dataSetPattern: true` | the named data set + `dataSetCondition`/`dataSetConditionType` |
 | `useExpression: true` | `sectionExpression` |

13. **Volume specification**: For each item, **at least one way to specify volume must be present**:
    - `volume` key (string, µL) — preferred for most items, or
    - `amount` and `amounts` — if per-probe volumes are needed (Span-8 specific)
    An **omitted** `volume` key defaults to `0` (which may cause technique validation errors). An **empty string** `""` is **not** the same as omitted — it fails at enqueue with `Invalid volume specified for <source|destination> N.` Every item that carries volume must give a non-empty numeric value or an evaluable expression; to leave it unset, omit the key rather than authoring `""`.

14. **Volume per source vs. per destination**: For a Transfer step, always set the volume on the destination.

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

18. **Leave tips vs. discard**: If `leaveTipsOn` is `true`, tips are left on the pod; if `false`, they are discarded to the location specified by `unloadLocation` (Fixed-8 only) or as defined in Instrument Setup (Multichannel, Span-8).

19. **Item count constraint**: `items` **must contain at least 2 elements** (one source, one destination).

20. **Item `labwareClass` must match (or be compatible with) the labware at `position`**: At enqueue the resolved labware class is checked against each item's `labwareClass`. A mismatch that is not a compatible substitute raises a class-mismatch error; an entirely unknown expected class raises a "labware class not found" error.

21. **`useCurrentTips` requires tips already on the pod**: If `useCurrentTips` is `true` but no tips are loaded on the pod when the step runs, enqueue fails with a "no tips loaded" error.

22. **Non-LLS tips are incompatible with LLS-enabled techniques (Span-8)**: The stock Span-8 pipetting techniques enable liquid level sensing on the initial descent, but non-LLS tip classes (e.g., `T190F`, `T50F` — filtered tips with `LLS: false` in the catalog) cannot liquid-level-sense. Combining them with the default auto-selected Span-8 technique hard-fails enqueue with a wrong-tip error. To use non-LLS tips, either pick an LLS-capable tip class or set `overrideHeight: true` on the item with a fixed-reference `heightFrom` (`1`/`"Bottom"` or `2`/`"Top"`) so the technique's initial LLS branch is bypassed.

23. **Volume tracking (cross-reference)**: The engine tracks a running per-well volume for the source and destination wells and rejects both over-dispense (fills exceeding the well/labware capacity) and under-aspirate (drawing more than the tracked source volume). Author source starting volumes and reservoir dead volumes in [Instrument Setup](03-Instrument-Setup.md) — see the "Volume tracking" note there.

24. **`numberOfTips` for Fixed-8 must match the pattern of selected wells**: For a Fixed-8 pod, the number of selected wells in a given column must be divisible evenly by `numberOfTips`. For example, to pipet to four rows in a destination, `numberOfTips` can be `1`, `2`, or `4`, but cannot be `3` or anything higher than `4`.

---

## Tip Budget

Tip consumption scales with pod family and `changeTips*` settings:

- **Fixed-8**: one tip-load per aspirate cycle draws `numberOfTips` tips (1-8). A whole-plate 96-well transfer with `numberOfTips: 8` and `changeTipsBetweenSources`/`changeTipsBetweenDests` at their defaults uses 96 tips (12 columns x 8).
- **Span-8**: one tip-load draws up to 8 tips (one per active probe from `useProbes` / `mandrelExpression`). A whole-plate 96-well transfer with all 8 probes active uses 96 tips.
- **Multichannel**: 96 or 384 tips per load, depending on the head.

Place enough tip boxes of the right class on the deck to cover the worst-case tip count for the method — or use `changeTipsBetween*: false` where cross-contamination is acceptable to reduce consumption.

---

## Multichannel: what a Transfer item can and cannot address

On a Multichannel pod a Transfer item's well selection resolves to **whole sections or quadrants**
of the labware, not to an arbitrary set of wells. The item-level `selectionInfo` is a list of
integer section/quadrant indices (see the `selectionInfo` row above and
[Multichannel Aspirate](20-Multichannel-Aspirate.md#enumerated-constrained-values) §Enumerated) — there is no
boolean 96-well mask on this pod family.

That is enough for whole-plate and whole-quadrant work, and not enough for a large class of
routine science:

- partial-plate layouts, where only some columns carry sample;
- control wells — an NTC, a no-RT control, reference wells — that must **not** receive what
  everything else receives;
- replicate patterning that reuses one tip load across a defined set of wells.

Authoring these as a plain Multichannel Transfer does not fail at enqueue. It silently widens the
operation to the whole section, which dispenses into the very wells the controls depended on
staying clean. A method can therefore be structurally valid and scientifically void.

For that work use the **Multichannel Select Tips** container family instead
([Multichannel Select Tips](26-Multichannel-Select-Tips.md) and the child
Aspirate/Dispense/Mix/Load/Unload steps, docs 27–35), which selects individual rows and columns of
tips and addresses wells accordingly.

Before authoring a Multichannel Transfer, check whether every well in the target section should
really receive the same thing. If the answer is no, this is not the step.

---

## Mixing in place

Mixing at a well is a **technique** property on every pod family, not an item-level Transfer key. An item-level `mixCount` on a Transfer item is inert (see the `mixCount` row in the item table) and does nothing at run time regardless of pod.

Multichannel additionally offers a standalone `Multichannel Mix` step and Fixed-8 offers `Fixed-8 Mix` when a mix is wanted without an accompanying transfer. Span-8 has no standalone Mix step — use a technique with a built-in mix.

The technique's mix volume goes through the same volume tracker as any aspirate, and which side the mix runs on decides what it is budgeted against: an Aspirate-side mix is checked before the transfer draws from the source, a Dispense-side mix after the delivery lands. See [Well Volume Tracking](../concept-guides/05-well-volume-tracking.md).

---

## Dispensing to waste

A Transfer dispenses to a labware position — there is no separate "liquid trash" destination for it, and the `Trash` characteristic (`<Any Trash>`, `WhenDone`, `DiscardTipsLocation`) routes **labware** and **tips**, not liquid. Liquid still in the tips is commonly discarded at a **wash station** instead: the Span-8 and Multichannel Wash Tips steps empty tip contents there (`dispenseTipContentsOnly`, or the passive `toWaste` volume), usually followed by a rinse that clears it — see [Span-8 Wash Tips](19-Span-8-Wash-Tips.md). Whether a particular liquid is appropriate to send through the wash station depends on the liquid and your instrument's plumbing and is outside the scope of these docs.

To discard bulk liquid **with a Transfer**, author a Transfer whose destination item names either a wash station or a labware position you have designated as waste — typically an empty reservoir placed on a free deck position for that purpose in Instrument Setup. From the step's point of view this is an ordinary destination: the item carries a normal `position`, `labwareClass`, `volume`, `pattern`/`selectionInfo`, and `prototype`; nothing about the destination is special-cased in the engine. If a wash station is selected as a destination, then the `labwareClass` must be the appropriate permanent labware (e.g. `"WashStation"` or `"WashStationSpan8"`).

---

## Structural Context

This step is a leaf and does not support subSteps.

---

## Canonical Examples

### Example 1: Simple Fixed-8 Transfer (single source to single destination, current tips)

```json
{
  "stepType": "Transfer",
  "parameters": {
    "activeWashTechnique": "",
    "autoSelectActiveWashTechnique": false,
    "changeTipsBetweenDests": false,
    "changeTipsBetweenSources": false,
    "dynamic?": true,
    "items": [
      {
        "autoSelectPrototype": false,
        "colsFirst": true,
        "customHeight": false,
        "dataSetPattern": false,
        "height": -2.0,
        "heightFrom": 0,
        "labwareClass": "BCFlat96",
        "liquidType": "Well Contents",
        "localPattern": true,
        "overrideHeight": false,
        "pattern": {
          "_biomekType": "comArray",
          "arraySubtype": "boolean",
          "values": [true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false]
        },
        "position": "P9",
        "prototype": "F8 Medium",
        "referencedPattern": "",
        "rowsFirst": false,
        "sectionExpression": "",
        "selectionInfo": {
          "_biomekType": "comArray",
          "arraySubtype": "boolean",
          "values": [true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false]
        },
        "setMark": true,
        "startAtMark": false,
        "startAtSelection": true,
        "useExpression": false,
        "volume": "10",
        "wellsX": 12
      },
      {
        "autoSelectPrototype": false,
        "colsFirst": true,
        "customHeight": false,
        "dataSetPattern": false,
        "height": 0.0,
        "heightFrom": 0,
        "labwareClass": "BCFlat96",
        "liquidType": "Tip Contents",
        "localPattern": true,
        "overrideHeight": false,
        "pattern": {
          "_biomekType": "comArray",
          "arraySubtype": "boolean",
          "values": [false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false]
        },
        "position": "P9",
        "prototype": "F8 Medium",
        "referencedPattern": "",
        "rowsFirst": false,
        "sectionExpression": "",
        "selectionInfo": {
          "_biomekType": "comArray",
          "arraySubtype": "boolean",
          "values": [false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false, false, true, false, false, false, false, false, false, false, false, false, false]
        },
        "setMark": true,
        "startAtMark": false,
        "startAtSelection": true,
        "useExpression": false,
        "volume": "10",
        "wellsX": 12
      }
    ],
    "leaveTipsOn": false,
    "mandrelExpression": "",
    "numberOfTips": "8",
    "pod": "Pod1",
    "podType": "Fixed8",
    "repeats": "1",
    "repeatsByVolume": false,
    "replicates": "1",
    "showTipHandlingDetails": true,
    "showTransferDetails": false,
    "solvent": "Water",
    "span8Wash": false,
    "span8WashVolume": "2",
    "span8WasteVolume": "1",
    "splitVolume": false,
    "splitVolumeCleaning": false,
    "stop": "Destinations",
    "tipLocation": "BC230",
    "useCurrentTips": true,
    "useDisposableTips": false,
    "useExpression": false,
    "useFixedTips": false,
    "useJIT": true,
    "useMandrelSelection": true,
    "useProbes": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true, true, true, true, true, true, true, true]
    },
    "wash": false,
    "washCycles": "3",
    "washVolume": "110%",
    "wizard": false
  }
}
```

## Common Mistakes

- **Per-item `pattern`/`selectionInfo` shorter than the labware's well count** — the `values` comArray must be full-length, not just the intended wells. Full-length means one entry per **addressable position** on that labware, which is not always 96: `wells X × wells Y` for a titer plate or tube rack (96 booleans for a 96-well plate), but the **section count** for a reservoir — `[true]` for a one-section `BCFullReservoir`, four entries for a `ModularReservoir`. Short arrays fail enqueue with a pattern-shape error.
- **Treating `podType` as pod-family selection** — pod family is resolved from the named `pod`. Setting `podType: "Span8"` on a Fixed-8 pod does not redirect the step to a Span-8 pod — but it is still wrong: keep `podType`/`span8` in sync with the resolved pod; see §Parameters row `podType`.
- **`autoSelectPrototype: true` on a Fixed-8 item**. Fixed-8 items must set `autoSelectPrototype: false` with a named `prototype`.
- **Forgetting to update `labwareClass`/tip-box after a deck-class change** — when the labware class on a position changes in Instrument Setup, every downstream `labwareClass`/`tipLocation`/`what` field must follow, or the step fails at enqueue with a class-mismatch error.
- **Wash enabled without a resolvable `solvent`** — the named solvent must resolve to a wash station on the deck; missing/misnamed stations surface only mid-run as a tip-cleaning error at the source position, not at authoring time.
- **`tipLocation` set to a tip class when the deck box carries a distinct instance name** — a class-only query only matches boxes whose instance name is empty or equal to the class name. If the deck's tip box has been renamed (its `properties.name` differs from the class), the tips-not-found error at Rule 6 fires — author the box's instance name or its deck position instead.
- **Omitting `changeTipsBetweenDests` on a Fixed-8 or Multichannel Transfer** — the default for those pod families is `false` (reuse tips across destinations), so omission yields the cross-contamination-risky behavior even though the same key defaults to `true` on Span-8. Author `changeTipsBetweenDests: true` whenever cross-contamination is not the intent; see the `changeTipsBetweenDests` row.
