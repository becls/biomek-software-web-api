# Pod Settings Format Specification

**What this document is.** A single spec for the **pod-settings** portion of a Biomek
**Instrument Settings** JSON export: what a pod-settings record looks like, and how to read facts
out of it. The umbrella export (`format:"Biomek Instrument Settings"`) bundles two categories
in one file — **pod settings** and deck layouts. This doc owns the **pod
settings** category (`podSettings`), plus the shared envelope and the `$id`/`$ref` reference
mechanics you need to read *any* part of the file. Part 1 defines the source format (envelope,
category keys, the pod record, value encoding, `$id`/`$ref`). Part 2 is the extraction recipe,
worked end-to-end against a sample export.

**The deck portion is documented separately.** Deck layouts (`deckLayouts`) live in the same
file but are specified — with their own conversion recipe — in
[Deck Layouts Format Specification](../deck-layouts/deck-layouts-format-spec.md). This
doc does not duplicate it; when you need decks, go there.

---

# Part 1 — the source format

## What pod settings are (read this first)

`podSettings` is the per-pod hardware configuration of the instrument: one entry per installed
pod, carrying that pod's **class** (Multichannel / Span-8 / Fixed-8), head type, axis travel limits,
calibration/gripper offsets, speed limits, and current tip state.

**Why this format is useful:**

- To learn **how many pods** the instrument has and **what each pod is** (Multichannel / Span-8 /
  Fixed-8) before authoring pod-specific steps.
- To read a pod's **axis limits**, calibration, and gripper offsets.
- To cross-check a Multichannel or Span-8 pod's identity against its human-readable `asText`
  summary. **Fixed-8's `asText` is a stub** (just the literal `"Fixed-8 pod settings"`), so
  do not rely on it for that class — read `settings` directly.

## Where pod settings come from: the Instrument Settings JSON export

Pod settings are **not** a standalone file. They live inside a **"Biomek Instrument Settings"**
JSON export.

It is produced by the Import/Export Utility's "Save As JSON" action: in the Editor,
*File > Export > Instrument Settings*, select folders in the instrument tree (Pod Settings /
Deck Layouts), *Export*, then *Save As JSON*.

### Contrast with method JSON

| | Method JSON (`Method-JSON-Structure.md`) | Instrument-settings JSON (this doc) |
|---|---|---|
| `format` value | `"Biomek Method"` | `"Biomek Instrument Settings"` |
| Purpose | Ordered tree of **steps** to run | The **machine**: pods and deck layouts |
| Top-level keys | `format`, `formatVersion`, `author?`, `description?`, `steps[]` | `format`, `formatVersion`, `podSettings?`, `deckLayouts?` |
| Value encoding | `_biomekType` provider stack | **Same** stack + instrument-only providers + `$id`/`$ref` refs |

There is no `steps`, `stepType`, or `subSteps` here. Do not look for a method envelope.

## Top-level envelope

```jsonc
{
  "format": "Biomek Instrument Settings",
  "formatVersion": "1.0",
  "podSettings":  { ... },
  "deckLayouts":  { ... }
}
```

| Key | Always present? |
|---|---|
| `format` | Yes — literal `"Biomek Instrument Settings"` |
| `formatVersion` | Yes — the **Instrument Settings** format version, currently `"1.0"` (independent of the Biomek Method JSON `formatVersion`) |
| `podSettings` | Only if Pod Settings were exported and non-empty |
| `deckLayouts` | Only if Deck Layouts were exported and non-empty (spec'd in the deck doc) |

Key points:

- The category keys are the camelCase renamings of the instrument-tree folder names:
  `"Pod Settings"→podSettings`, `"Deck Layouts"→deckLayouts`. The original spaced names never appear in the JSON.
- **A category appears only if the user selected it AND the instrument has something for it.**
  You can export just one folder, and empty categories are dropped. So a file may have `podSettings` and
  no `deckLayouts`, or vice-versa.
- **First step when reading any file:** check `format`, then list which of the category
  keys are present. That tells you what the export covers before you dive in.

## The pod-settings category (`podSettings`)

`podSettings` is a JSON object **keyed by pod name** (`pod1`, `pod2`, …). Each entry has exactly
**three** top-level fields:

| Field | Type | Meaning |
|---|---|---|
| `asText` | string | Human-readable summary of the pod. On **Multichannel** and **Span-8** it carries validation time, head type, axis ranges — a useful cross-check. On **Fixed-8** it is a stub (just `"Fixed-8 pod settings"`) with none of that; read `settings` directly for Fixed-8. |
| `settings` | object | The structured pod record — class, head type, limits, offsets, tips (see below). |
| `timestamp` | object | A `_biomekType:"dateTime"` object (see [Timestamps](#timestamps)) marking when this settings snapshot was **taken**, i.e. export time — not when the pod was last configured. |

**Not every key is necessarily an installed pod.** The pods physically installed on the instrument
are named `pod1` and `pod2`. If the operator has *saved pod settings* under their own names in
Hardware Setup, those saved profiles are exported into the same `podSettings` object, keyed by
their saved names (letters, digits and underscores only) with the same camelCase key policy
applied. A saved profile has the same three-field shape as an installed pod, so structure alone
will not distinguish them — **identify installed pods by the `pod1`/`pod2` names rather than by
counting keys.** None of the seven default instrument exports contains a saved profile, so
the examples below all show the installed-pod case.

### The pod class: read `settings.type`, and know the two-key nuance

- **`settings.type`** is the **pod class string** — one of **`Pod96`**, **`Pod8Span`**, or
  **`Fixed8`**. *This* is what tells you what the pod is; read it first.
- **`settings.podType`** is a **head-family code**: `MC` (multichannel), `SPAN8`, or `Fixed8`.
  For Multichannel and Span-8 it is a *different* value from `type` (`MC` vs `Pod96`, `SPAN8` vs
  `Pod8Span`). For **Fixed8** the two collapse — both fields carry `"Fixed8"`. It is present on
  every pod, which is why treating `settings.podType` as guaranteed-present is safe across pod
  classes.

| Pod class | `settings.type` | `settings.podType` |
|---|---|---|
| Multichannel (96-head OR 384-head) | `Pod96` | `MC` |
| Span-8 | `Pod8Span` | `SPAN8` |
| Fixed-8 | `Fixed8` | `Fixed8` |

### The `settings` field set — per pod class

`settings` is a plain dictionary (no `_biomekType`). The key set differs substantially by pod class,
so a doc built around only one class will mislead readers of another. The table below is grounded
in seven representative instrument configurations (see [Representative instrument configurations](#representative-instrument-configurations)).

**Common to every pod class** (13 keys):

| Key | Meaning |
|---|---|
| `type` | Pod class string (`Pod96` / `Pod8Span` / `Fixed8`) |
| `podType` | Head-family code (`MC` / `SPAN8` / `Fixed8`) |
| `cmdGen` | Command-generator identity for this pod |
| `additionalSafeHeight` | Safe-height margin above deck for moves (cm) |
| `alwaysMaxZ` | If true, always raise to Z-max between motions |
| `currentSpeed` | Current speed setting — a percentage, clamped by the pod to `speedLimit` |
| `speedLimit` | Maximum allowed speed as a **percentage** (0–100) of the axis maxima, never mm/s. The pod's `asText` renders it as e.g. `Speed Limit: 100%`. The other speed fields (`discardSpeed`, `tipLoadSpeed`, `tipUnloadShuckSpeed`) are percentages too. |
| `rT_CurrentSpeed` | Runtime-state mirror of `currentSpeed` |
| `gripperXOffset`, `gripperYOffset`, `gripperZOffset` | Gripper calibration offsets (cm). These are variant-typed, so a whole-number value may arrive without a decimal — Fixed-8 exports carry `"gripperXOffset": 0` while 96-head exports carry `"gripperYOffset": 0.0`. Accept both forms (see [Value encoding](#value-encoding-shared-stack--instrument-extensions)). |
| `max`, `min` | Per-axis travel limit dictionaries — see [Axis limits](#axis-limits-max--min) |

**Multichannel + Span-8 shared** (5 keys, absent on Fixed-8):

| Key | Meaning |
|---|---|
| `additionalSafeHeightGripper` | Extra Z margin when the gripper is deployed |
| `gripperModel` | Gripper model identifier |
| `headSerialNumber` | Serial number of the mounted head assembly |
| `sensorEnabled` | Whether the pod's tip-presence / other sensor is enabled |
| `timeoutSlop` | Motion timeout tolerance |

**Fixed-8 + Span-8 shared** (17 keys, absent on Multichannel — these are the per-probe fields):

| Key | Meaning |
|---|---|
| `axes` | Motion-parameters nested dictionary. **Both the keying and the inner field set differ by class** — see [Axis motion parameters](#axis-motion-parameters). |
| `tip1` … `tip8` | Per-probe tip descriptors — see [Per-probe tips and tip state](#per-probe-tips-and-tip-state) |
| `rT_Tip1` … `rT_Tip8` | Runtime per-probe tip state (same shape as `tipN`) |

**Pod96 (Multichannel) only** (15 keys):

| Key | Meaning |
|---|---|
| `headType` | Head type code |
| `headTypeLabel` | Human-readable head label (e.g. `"300 µL MC-96 Head"`) |
| `framingProbe` | Reference probe used for head framing |
| `perActionOverhead` | Per-motion time overhead (calibration constant) |
| `tips` | Tip descriptor for the whole 96-head (single collection, not per-probe) |
| `rT_Tips` | Runtime tip state for the whole head |
| `tipSeatingDepth` | Depth to press tips into the labware for seating |
| `tipLoadForceLimitOffset`, `tipLoadForceLimitSlope` | Tip-load force-limit calibration |
| `tipLoadPushOffDistance` | Distance the head backs off during tip load |
| `tipLoadSettlingTime` | Wait time after tip pickup |
| `tipLoadSpeed` | Z-axis speed during tip pickup |
| `tipLoadXRangePadding` | X-axis padding at tip-load boundaries |
| `tipUnloadShuckSpeed`, `tipUnloadZOffset` | Tip-unload motion parameters |

#### Which head is mounted

Two export fields identify the head, and a third gives its volume ceiling as a number:

- **`settings.headType`** (code) and **`settings.headTypeLabel`** (human) name the head. The µL
  figure in the name is the head's plunger stroke — a `"300 µL MC-96 Head"` tops out at 300 µL.
- **`settings.max.d`** is that same figure as a number, in µL, and is the value to read. It matches
  the mounted head's rating in every configuration in the sample exports, but importing a saved
  pod-settings file can overwrite it, so read the export rather than trusting the label.

What that ceiling is measured against, and why it means something different on the other two pod
classes, is in [What `max.d` means, and when it is not the pipetting
ceiling](#what-maxd-means-and-when-it-is-not-the-pipetting-ceiling).

Read the head and its ceiling for each MC pod straight from the export:

```
jq -r '.podSettings | to_entries[]
        | select(.value.settings.podType=="MC")
        | "\(.key): \(.value.settings.headTypeLabel) → ceiling \(.value.settings.max.d) µL"' \
   biomek-i7-hybrid.json
# pod1: 300 µL MC-96 Head → ceiling 300 µL
```

**Fixed-8 only** (2 keys):

| Key | Meaning |
|---|---|
| `gripper` | Fixed-8 pod's integrated gripper descriptor |
| `podSerialNumber` | Serial number of the Fixed-8 pod |

**Pod8Span (Span-8) only** (13 keys):

| Key | Meaning |
|---|---|
| `syringes` | Per-probe syringe configuration (size and state) |
| `probeSize` | Probe geometry / size class |
| `probesSFOffset` | Scale-factor offset per probe |
| `loadingOffsets` | Per-probe Z-offset used at tip load |
| `dAxisCycles` | Cycle count for the D (dispense) axes |
| `discardSpeed` | Speed used to discard tips |
| `tipLoadTime` | Wait time during tip load |
| `detectTips` | Whether tip-presence detection is enabled |
| `brokenTips` | Per-probe broken/damaged flag mask |
| `postRunWashVolume` | Volume used to wash probes after a run |
| `systemTrailingAirgap` | System-level trailing air-gap volume |
| `calibrateVolumeForProbes` | Whether per-probe volume calibration is enabled |
| `lastValidation` | Timestamp of last validation run |

### Axis limits (`max` / `min`)

`max` and `min` are per-axis dictionaries. **The axis set varies by pod class**, not just X/Y/Z:

- **Pod96**: `x`, `y`, `z`, `d` (dispense) plus gripper axes `gg`, `gr`, `gy`, `gz`.
- **Pod8Span**: adds per-probe travel limits `z1`..`z8` and `d1`..`d8`, plus a `span` axis (probe
  spacing), and an internal `zMaxMC` limit used when the Span-8 must reach an MC head's zone.
  `zMaxMC` appears in `max` **only** — `max` and `min` are not guaranteed to have the same key
  set, so do not iterate one and index the other blindly.
- **Fixed-8**: `x`, `y`, `z`, `d` **only** — the pod moves as a single block, so there are no
  per-probe axis limits, and **no gripper axes**. A Fixed-8 with an integrated gripper still shows
  only `d`/`x`/`y`/`z` here: the gripper is described by the top-level `gripper` block and is
  driven through the **D axis**, using the gripper offsets/angles inside `cmdGen.conversion`
  (`gripperDegreesPerDAxisTravel`, `gripperFingerLength`, `restingGripperAngle`, …).

A consumer looking only at `x/y/z` will silently miss the gripper and per-probe limits — always
enumerate the actual keys under `.settings.max` / `.settings.min` for the pod at hand. The
`asText` summary is **not** a substitute: even on a 96-head pod its "Axis Limit Settings" block
lists only X, Y, Z and D.

#### What `max.d` means, and when it is not the pipetting ceiling

`d` is not a distance. On every pod class the D axis is denominated in **µL**: its position is the
plunger or syringe position expressed as the volume currently drawn. So `max.d` is a volume — but
*which* volume depends on the pod class, and on one of the three it is not a pipetting limit at all.

| Pod class | What `max.d` is | The pipetting ceiling? |
|---|---|---|
| Pod96 (Multichannel) | The mounted head's plunger stroke | Yes |
| Pod8Span | The installed syringe's capacity, per probe (`max.d1`…`max.d8`; `max.d` mirrors probe 1) | Yes |
| Fixed-8 | The far end of the D axis, where the **gripper** is closed | **No** — see below |

Where it is the ceiling, the check is on the **plunger position the move ends at**, not on the
move's own volume, so whatever the tips already hold counts against it: a 200 µL aspirate on top of
150 µL already in the tips overruns a 300 µL head. It most often bites when a *computed* volume — a
base volume plus conditioning, excess, or reagent terms — pushes one aspirate over the top. The
overrun is caught at enqueue, before any motion, and the two families word it differently:

```
Move of <pod> to D of <n> µL violates D axis maximum of <max> µL.
Move of <pod> syringe #<n> to <v> µL violates the syringe maximum of <max> µL.
```

The first is Multichannel. The second is Span-8, and it names the offending **probe**, because
Span-8 limits are per-probe — read `max.d1`…`max.d8` rather than assuming one value covers the pod.

**Fixed-8 is the exception.** One motor drives both the plunger and the gripper, so the D axis runs
past the pipetting range and on into the gripper's travel. `max.d` is the far end of that combined
travel — on the sample i3 export, roughly a third above the largest volume the pod can actually
pipette. Past the pipetting range the pipetting seals disengage, so the pipetting path enforces its
own, lower maximum and rejects anything beyond it at enqueue:

```
The D axis position of <n> µL is outside the supported pipetting range of the pod.
```

**That lower maximum is not in the export**, under any key: the export carries the settings that are
editable in Hardware Setup, and this one is not among them. There is therefore no way to compute a
Fixed-8 pod's true pipetting ceiling from an instrument settings file. Do not read `max.d` as the
ceiling on a Fixed-8 pod, and do not present it to a user as one — in practice a Fixed-8 tip's
capacity is well below either number, so tip capacity is the limit a method meets first.

Note that for a Span-8 pod, the system trailing air gap must be accounted for in
D axis movements. For example, when using a 1mL syringe size, the maximum draw
of the D axis is 1mL minus the system trailing air gap amount.

### Axis motion parameters

`axes` (Fixed-8 and Span-8 only) holds the per-axis motion profile — velocity, acceleration, jerk
and their resolutions. **Both the keying and the inner field set differ by pod class**, so
enumerate the inner keys instead of assuming a fixed set:

| | Span-8 | Fixed-8 |
|---|---|---|
| Outer keys | Per probe: `z1`…`z8`, `d1`…`d8` | Whole pod: `x`, `y`, `z`, `d` |
| Velocity keys | `vmax`, `vmin`, `vres` (all lowercase) | `vMax`, `vMin`, `vres` (note the capital M) |
| Acceleration keys | `amax`, `amin`, `ares` | `amax`, `amin`, `ares` |
| Jerk keys | `jmax`, `jmin`, `jres` — **z-axes only** | `jmax`, `jmin`, `jres` — on every axis |
| Extra keys | `startDelay` on z-axes; `backlash` and `vstart` on d-axes | none |
| Always present | `setupTime` | `setupTime` |

The mixed casing on Fixed-8 (`vMax` next to `amax`) is correct and intentional: each JSON key is
the camelCase rendering of the key as it is stored, and the stored names really are `VMax`/`Amax`.
Do not normalize one family's spelling into the other's.

### Per-probe tips and tip state

On Fixed-8 and Span-8, `tip1`…`tip8` (and their runtime mirrors `rT_Tip1`…`rT_Tip8`) describe
what is on each probe. Two things trip up readers:

- **They are `null` when nothing is mounted.** A Fixed-8 export and a Span-8 export configured for
  disposable tips both show `"tip1": null` … `"tip8": null` and `"rT_Tip1": null` …; a Span-8 with
  fixed tips installed shows populated objects. `null` means "no tip on this probe", not "field
  missing".
- **The tip discriminator is not on the outer object.** When populated, `tipN` contains a single
  `class` sub-object. `class` carries the tip-class scalars (`name`, `capacity`, `filtered`,
  `lls`, geometry) and usually a `$id`, because it is shared. The `_biomekType:"tip"` marker lives
  one level further down, on `class.template`, whose own `class` key is a `$ref` back to the
  enclosing class. Looking for `_biomekType:"tip"` at the top of `tipN` will always miss it.

For a Pod96 there are no per-probe tip fields at all — the whole head has one `tips` / `rT_Tips`
value instead (also `null` when no tips are loaded).

### Fixed vs. disposable probes

**The discriminator is `tipN.class.fixed`, not whether `tipN` is populated.** `null` only means "no
tip on this probe right now", which is the normal resting state of a *disposable* probe. A probe
configured for **fixed** tips carries a populated descriptor whose class has `"fixed": true` — this
is what `biomek-i5-span8.json` shows on all eight probes, with the class named `Fixed100`. A
disposable-tip Span-8 export instead shows `null` (or, if tips happened to be loaded at export
time, a class with `"fixed": false`). Compare `biomek-i7-span8.json`, the same instrument with the
tip configuration swapped to disposable.

**The instrument's probe configuration wins; a method cannot override it.** The pod reads each
probe's fixed/disposable state from that per-probe tip object's class `Fixed` flag when it
initializes, and no method JSON key feeds into that read. A protocol's tip choice does not change
probe hardware — it is only checked against it:

- **Loading disposable tips onto a fixed probe fails at enqueue** with `Cannot load tips on fixed
  tips.` — raised for any selected probe that is configured fixed. Against an all-`Fixed100`
  export like `biomek-i5-span8.json` this means *every* probe, so no probe selection avoids it.
- **Unloading silently skips fixed probes** rather than failing: the discard mask is ANDed with
  "not fixed" and "has a tip", so an unload against an all-fixed pod is a no-op.
- The **Span-8 Load Tips / Unload Tips steps are offered at all** only when at least one Span-8
  probe is non-fixed, which is why a disposable-tip method authored against a fixed-tip instrument
  looks valid on paper and fails on the machine.

**How to request disposable tips against a fixed-tip export:** you cannot, from method JSON. The
probe configuration is instrument state, changed in Hardware Setup (and, on real hardware, by
physically swapping the mandrels) — not by any key in a method. Either run the method on an
instrument whose Span-8 probes are configured disposable, or change that instrument's probe
configuration first. Read the target instrument's `podSettings.<pod>.settings.tip1…tip8` before
authoring, and treat an all-`fixed: true` Span-8 as incompatible with a disposable-tip protocol.
The pod's `asText` carries the same fact in readable form — `Probe Configurations: 1: Fixed100
tips, 1 mL syringes.`

### Runtime-state (`rT_*`) fields

The `rT_` prefix marks **runtime state** — a live snapshot at export time, not static
configuration. The name varies by pod class:

- **Pod96** → `rT_Tips` (single tip collection for the whole 96-head).
- **Pod8Span, Fixed-8** → `rT_Tip1` … `rT_Tip8` (per-probe).
- **All classes** also carry `rT_CurrentSpeed`.

Treat every `rT_*` value as "current at export time" — do not read one as fixed configuration.

### Timestamps

`timestamp` is a `_biomekType:"dateTime"` object, not a bare string:

```jsonc
"timestamp": { "_biomekType": "dateTime", "value": "2026-08-20T13:49:57.0670000" }
```

Read the ISO-8601 string from `.value`. The same encoding applies to any other dateTime in the
file.

**What the pod `timestamp` actually means.** It is generated when the export is produced, not when
the pod was last edited in Hardware Setup: each pod's settings snapshot is stamped as it is
extracted, so in a two-pod export `pod1` and `pod2` typically differ by a millisecond or two. Use
it to date the *export*; do not read it as a configuration-change time. (`lastValidation` on a
Span-8 is a different, genuinely persisted timestamp.)

## The other category (brief)

- **`deckLayouts`** — decks defined on the instrument. **Fully specified, with its own JSON→map
  conversion recipe, in [Deck Layouts Format Specification](../deck-layouts/deck-layouts-format-spec.md).**
  Not repeated here.

## Value encoding (shared stack + instrument extensions)

Values are encoded exactly as in the method format, so nothing here is
instrument-specific except the extra aggregate types listed below. The rules you need to read a
pod-settings record are repeated in full here; the same rules are covered from the authoring side
in [Method JSON Structure §"Parameter Value Encoding"](../../Method-JSON-Structure.md#parameter-value-encoding) / §"JSON Parsing Behavior".

**Primitive encoding**

- Strings, booleans and `null` are ordinary JSON primitives. Numbers are written
  locale-invariantly (always `.` for the decimal separator, never a thousands separator).
- **Numbers carry their COM variant type in their spelling.** A floating-point value always gets a
  decimal point, even when whole (`10` → `10.0`). An integer value is written bare (`100`).
  Because instrument values are variant-typed, *the same field can be a float on one pod class and
  an integer on another* — `gripperXOffset` is `0.29129999999999967` on a 96-head pod and a bare
  `0` on Fixed-8. Parse every numeric field as a number and never key off the presence of a
  decimal point.
- Floats are written at full round-trip precision — expect values like `6.375400000000001`. Do not
  assume the exporter rounds.
- A JSON object with **no** `_biomekType` is a plain nested dictionary.

**`_biomekType` discriminators from the shared stack**, all of which can appear in an instrument
export:

| `_biomekType` | Shape | Meaning |
|---|---|---|
| `"dateTime"` | `{ "_biomekType": "dateTime", "value": "<ISO 8601>" }` | A date/time. `value` is the ISO 8601 round-trip form, e.g. `2026-08-20T13:49:57.0670000`. |
| `"empty"` | `{ "_biomekType": "empty" }` — no other keys | The COM "empty" variant. Distinct from JSON `null`, which is the COM "null" variant. |
| `"missing"` | `{ "_biomekType": "missing" }` — no other keys | The COM "missing"/parameter-not-supplied variant. Also distinct from `null`. |
| `"nonFiniteNumber"` | `{ "_biomekType": "nonFiniteNumber", "value": "NaN" }` | A non-finite float. `value` is the string `Infinity`, `-Infinity`, or `NaN` — JSON has no literal for these. |
| `"comArray"` | `{ "_biomekType": "comArray", "arraySubtype": "double", "values": [ ... ] }` | A COM array, written as an object so its element type survives. `arraySubtype` is one of `boolean`, `byte`, `dateTime`, `double`, `integer`, `short`, `single`, `string`, `variant`. Optional `dimensionCount` (only when > 1) and `lowerBound` (only when non-zero) describe multidimensional arrays; `values` nests one array per dimension. |
| `"labware"`, `"technique"`, `"SILASMessage"` | see the method structure doc | Method-domain values that can be reached from instrument state. |

A plain JSON **array** is an ordinary Biomek list, not a COM array — the two are different types
and are spelled differently on purpose.

**Instrument-only `_biomekType` values**:

| `_biomekType` | What it is |
|---|---|
| `"device"` | A device — also carries `_deviceType` = aggregate class name, normally a ProgID (see the `device` field schema below for the observed value). |
| `"conversion"` | Deck↔pod geometry / gripper conversion block. |
| `"deckPosition"` | A deck position (see the deck spec). |
| `"tip"` | A tip / tip-class definition. |

An object aggregated with a class **not** in this table makes the export **fail loudly** rather
than silently degrade — so if a file exists, all its
aggregate objects were representable.

Field schemas for the instrument-only aggregates other than `deckPosition` (which is fully
covered in the deck spec):

- **`device`** — appears on deck positions wired to a hardware device (trash and similar
  fixtures). Carries the standard `_biomekType:"device"` plus a `_deviceType` string
  identifying the device's aggregate class name — normally a ProgID. The only value present in
  the exports included with Biomek is on the trash devices attached to `TR*` positions:

  ```json
  { "_biomekType": "device", "_deviceType": "Biomek5.TrashDevice" }
  ```

  Other device classes (including devices not implemented in Biomek itself) can appear. Compare
  the string whole and treat it as an opaque identifier — do not build behavior on how the ProgID
  is spelled. Any additional keys under a `device` object are internal wiring state; treat them as
  opaque.

- **`conversion`** — a block encoding a deck↔pod geometry / gripper
  correlation table (used when reconstructing pod-relative coordinates). Its inner keys
  are geometry/calibration coefficients tuned per pod install. **Treat as opaque**: it plays no
  part in method authoring — do not read its fields.
  Note that the `_biomekType:"conversion"` marker is **not** universal: 96-head and Span-8 pods
  carry it on `settings.cmdGen.conversion`, but the Fixed-8's equivalent block is a plain
  dictionary with no `_biomekType` at all. Locate the block by path, not by discriminator.

- **`tip`** — a tip-class definition. It is not the per-probe wrapper
  under `settings.tip1`…`tip8`; it is what those wrappers bottom out in, at
  `settings.tipN.class.template` (see [Per-probe tips and tip state](#per-probe-tips-and-tip-state)).
  Carries the tip class name and its behavioral
  scalars (capacity, filtered flag, LLS capability, geometry). The `LLS` scalar records whether
  the tip class itself is LLS-capable; **liquid-level sensing is only supported by the Span-8
  pod** — Fixed-8 and Multichannel pods do not perform LLS, regardless of the tip's `LLS` flag.
  Prefer the same tip-class facts
  from the project-items export ([`../project-items/`](../project-items/)) when authoring;
  the copy inside pod settings is a runtime snapshot.

If you encounter an unfamiliar key under any of these aggregates, treat it as opaque
rather than inventing a meaning — the categories above are not exhaustively field-schemad
because their contents are internal instrument state, not method-authoring input.

## References: `$id` / `$ref` — you MUST resolve these

The instrument object graph is **cyclic and full of shared objects** (a pod settings object may
be reached from two places; a deck position points at its group). JSON cannot nest a cycle, so
the serializer uses JSON-Reference metadata:

- An object reached **once** is written **inline** with no metadata.
- An object reached **more than once** is written **in full at its first occurrence** with an
  added `"$id"`, and **every later occurrence becomes** `{ "$ref": "<pointer>" }`.
- `$id`/`$ref` values are **JSON-Pointer-style strings** rooted at `#`, each segment URL-escaped
  and camelCased (space→`%20`, `/`→`~1`, `~`→`~0`), with numeric list indices.
  A real pointer looks like `#/podSettings/pod1/settings/tip1/class`. The escaping is **full** URL
  escaping, not just spaces — `#`→`%23`, `?`→`%3F`, `%`→`%25`, `é`→`%C3%A9` — so decode with a
  general URL-decoder rather than special-casing `%20`. Pod keys themselves never
  contain a space, so `%20` will not appear in a pod pointer — but
  implement the unescaping anyway: property names deeper in the graph do contain spaces (e.g. the
  deck-position `characteristics` keys `can Discard Tips` and `disallow AutoTeach`), and so can
  names in the other two categories.
- References can **cross categories** — the same object can be reachable from both `podSettings`
  and `deckLayouts`, and the second occurrence becomes a `$ref` into the first.
- Only **objects** carry reference metadata. A shared **array** is written
  inline at each occurrence (duplicated), never as a `$ref`.
- `$id`/`$ref` are **reserved** — real payload never uses those keys.

**How to resolve a `$ref` while reading:** treat the pointer as a path from the document root
(`#`), split on `/`, URL-decode each segment (`%20`→space, `~1`→`/`, `~0`→`~`), match keys
case-insensitively (they are camelCased), and read the object carrying the matching `$id`. E.g.
`#/podSettings/pod1/settings` = the `settings` object under `pod1`. An interpreter that does not
follow `$ref` will read incomplete records.

---

# Part 2 — reading pod settings out of the export

This is the extraction recipe. It uses `jq` paths for concreteness; `jq` does **not** follow
`$ref`, so when a value is `{ "$ref": "#/…" }` read the target path directly (see the
resolution rule above).

> **Worked-example note.** The snippets and values below are from a **sample i7-Hybrid export**
> (two pods: Pod1 = 96-head, Pod2 = Span-8). Use the values as *representative* shapes; field
> names carry over directly from the live object graph, but per-instrument values differ. See
> [Representative instrument configurations](#representative-instrument-configurations) below for the
> roster.

## Step 1 — confirm the envelope and list categories

```bash
jq -r 'keys[]' <export>.json
# → deckLayouts, format, formatVersion, podSettings
jq -r '.format, .formatVersion' <export>.json
# → "Biomek Instrument Settings"   "1.0"
```

If `podSettings` is absent, that folder was not exported (or the instrument had nothing) — it
does **not** mean the machine has no pods.

## Step 2 — enumerate pods and read each pod's class

`podSettings` is keyed by pod name. Read the **class** from `settings.type` (not `settings.podType`):

```bash
jq -r '.podSettings | to_entries[]
       | "\(.key): type=\(.value.settings.type)  podType=\(.value.settings.podType)"' \
  <export>.json
# pod1: type=Pod96     podType=MC      (example)
# pod2: type=Pod8Span  podType=SPAN8   (example)
```

Sample result: two pods — `pod1`=`Pod96`, `pod2`=`Pod8Span` — matching the i7-Hybrid mapping
(Pod1 = 96-head, Pod2 = Span-8). Read the installed pods from the `pod1`/`pod2` entries; any other
key is a saved pod-settings profile, not an installed pod (see
[The pod-settings category](#the-pod-settings-category-podsettings)).

Cross-check with the human summary — often the fastest sanity check on Pod96 and Pod8Span:

```bash
jq -r '.podSettings.pod1.asText' <export>.json
# Pod96  → "Validation Time: Not Validated … Head Type: 300 µL MC-96 Head … X: [ 10.52 , 108.27 ] …"
# Pod8Span → "Validation Time: Not Validated … Probe Configurations: 1: Fixed100 tips, 1 mL syringes. …"
# Fixed-8 → "Fixed-8 pod settings"   (stub — read settings directly instead; note Fixed-8 has no headType)
```

## Step 3 — read a pod's structured settings

```bash
jq -c '.podSettings.pod1.settings
       | {type, podType, headTypeLabel, min, max,
          gripperXOffset, gripperYOffset, gripperZOffset, speedLimit}' \
  <export>.json
```

- `type` → pod class; `headType`/`headTypeLabel` → mounted head.
- `min`/`max` → axis travel limits; `gripper*Offset` → gripper calibration.
- `rT_*` fields (e.g. `rT_Tips`) are **runtime state**, current at export time — don't treat them
  as static configuration.

## Step 4 — read the export timestamp

```bash
jq -r '.podSettings.pod1.timestamp.value' <export>.json
# → 2026-08-20T13:49:57.0670000
```

(`timestamp` is a `_biomekType:"dateTime"` object; the ISO string is under `.value`. It records
when the export was produced, not when the pod was last configured — see
[Timestamps](#timestamps).)

## Worked example — an i7-Hybrid pod-settings record (representative)

A single pod entry looks like this (trimmed to the load-bearing fields):

```jsonc
"pod1": {
  "asText": "Validation Time: Not Validated … Head Type: 300 µL MC-96 Head … X: [ 10.52 , 108.27 ] …",
  "settings": {
    "type": "Pod96",          // ← pod class: read THIS
    "podType": "MC",          // ← pod-family code, NOT the class
    "headType": "MC96_300",   // ← head code
    "headTypeLabel": "300 µL MC-96 Head",
    // Note the gripper axes: a reader that only looks at x/y/z misses gg/gr/gy/gz.
    "min": { "d": -82.66380166, "gg": 6.039,  "gr": -370.0, "gy": 15.834,
             "gz": 6.943,       "x": 10.52,   "y": 14.596,  "z": 10.996 },
    "max": { "d": 300.0,        "gg": 15.739, "gr": 190.0,  "gy": 62.683,
             "gz": 37.949,      "x": 108.2689, "y": 59.591, "z": 39.863 },
    "gripperXOffset": 0.29129999999999967, "gripperYOffset": 0.0,
    "gripperZOffset": -7.279999999999999,
    "speedLimit": 100.0,       // ← percent, not mm/s
    "rT_Tips": null            // ← runtime tip state; null = no tips loaded
  },
  "timestamp": { "_biomekType": "dateTime", "value": "2026-08-20T13:49:57.0670000" }
}
```

→ Read as: **Pod1 is a `Pod96` (96-channel) pod with a 300 µL MC-96 head**, snapshotted
2026-08-20. The companion `pod2` in this sample is a `Pod8Span`. Values shown are from a sample
i7-Hybrid export — regenerate against your own file for real numbers.

## Reading pitfalls (common mistakes)

- **Using `settings.podType` for the pod class.** It is a pod-family code (`MC`/`SPAN8`), not
  the class. Read **`settings.type`** (`Pod96`/`Pod8Span`/`Fixed8`).
- **Not following `$ref`.** Shared/cyclic objects live at their `$id` site and appear elsewhere
  as `{ "$ref": … }`. `jq` won't follow them — resolve the pointer yourself. Shared **arrays**
  are duplicated inline, not ref'd.
- **Treating `rT_*` fields as static config.** They are runtime state, current at export time.
- **Assuming a missing category means a missing capability.** `podSettings`/`deckLayouts`
  are dropped when not selected or empty — absence is not evidence.
- **Trying to import the file.** It is export-only (`FromJson` throws); treat it as a read-only
  snapshot.
- **Looking for decks here.** Deck layouts are in the same file but specified in
  [Deck Layouts Format Specification](../deck-layouts/deck-layouts-format-spec.md).

---

## Representative instrument configurations

The pod-settings tables in this document are grounded in seven representative instrument
configurations, each defined by a `.bif` instrument-configuration file. They are a sample chosen to
cover the pod and chassis combinations — **not an exhaustive catalog**. A real instrument can vary in
ways the export reflects: an i5 Span-8, for example, can carry either fixed or disposable tips, and
the roster shows just one such variant per row. Use it to find a configuration close to the
instrument you are authoring for, then read the authoritative values from that instrument's own
export. Re-exporting the same instrument reproduces the same structure and the same values, with one
exception: each pod's `timestamp` is stamped at export time, so it will differ (see
[Timestamps](#timestamps)).

| Configuration | Instrument | Pods | Deck layouts |
|---|---|---|---|
| i3 | i3 (Fixed-8) | `pod1` = `Fixed8` | `i3`, `standard` |
| i5 Multichannel | i5 Multichannel | `pod1` = `Pod96` (MC) | `multichannel`, `standard` |
| i5 Span-8 | i5 Span-8 | `pod1` = `Pod8Span` (SPAN8) | `span8`, `standard` |
| i7 Multichannel | i7 Multichannel (single MC pod) | `pod1` = `Pod96` (MC) | `dualMultichannel`, `hybrid`, `multichannel`, `span8`, `standard` |
| i7 Span-8 | i7 Span-8, tip-config swapped to disposable | `pod1` = `Pod8Span` (SPAN8) | same as i7 Multichannel |
| i7 Hybrid | i7 Hybrid (MC + Span-8) | `pod1` = `Pod96` (MC), `pod2` = `Pod8Span` (SPAN8) | same as i7 Multichannel |
| i7 Dual-MC | i7 Dual-MC, head types differ across pods | `pod1` = `Pod96`, `pod2` = `Pod96` | same as i7 Multichannel |

Notes on provenance:

- **All i7 configurations carry the same five default deck layouts** (`dualMultichannel`, `hybrid`,
  `multichannel`, `span8`, `standard`) regardless of which pod configuration is installed. The
  simulated pod-type change was made in Hardware Setup only; the default/standard i7 deck was not
  swapped even though a real hybrid deployment might do so. Treat the deck list as
  instrument-class geometry, not pod-derived.
- **The i7 Span-8 configuration** differs from a nominal i7-Span baseline only in the pod's
  tip configuration (disposable rather than fixed).
- **The i7 Dual-MC configuration** has two `Pod96` pods; their `headType` /
  `headTypeLabel` differ.
- i5 and i3 configurations carry only their two native decks each; i7 configurations carry all
  five because the chassis supports every layout.
