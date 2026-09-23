# Catalog: Techniques

This document describes only the default techniques that are included with
Biomek Software. A given Biomek system likely has additional techniques
developed by users which are preferred for use over the defaults. Users should
provide the names of available techniques on their systems to LLMs for use in
method authoring.

> **The names here are real and usable; only the JSON *format* is forward-looking.** Biomek has
> **no JSON export for project items** today, so the payloads below are the planned/future shape of
> the technique-provider JSON — not authored files. But the catalog documents the **actual default
> pipetting techniques included with Biomek** — the named entries in the project's Technique Browser
> that a pipetting step selects via `prototype` or auto-select — and these are the valid technique
> names to reference. A specific project may add its own, so treat this as the default set, not an
> exhaustive list.
>
> Unlike the other project-item catalogs, each technique here is shown in the
> **real serialized JSON shape Biomek produces for a technique** —
> the same `_biomekType: "technique"` payload that appears when a pipetting step embeds an
> inline `customPrototype`. `hardwareCompatibility` is **pod-based** (Multichannel /
> Span-8 / Fixed-8); chassis size does not affect it, and a hybrid instrument supports the
> union of its pods. Techniques are pod-family-specific by name convention (`F8 …` /
> `MC …` / `S8 …`) and by their internal `context.pod` list, so no default
> technique reaches all three pods and `["any"]` never appears here.

## Scope and derivation

- **Category:** Pipetting Techniques. Techniques are *not* one of the formal project-item
  categories the default projects carry alongside labware classes, labware patterns,
  liquid types, tip classes, and pipetting templates; they live in the project's technique
  library instead, reachable in the Biomek software via **Utilities > Technique Browser**
  and **Utilities > Technique Editor**, and are surfaced in method JSON either by name
  (`prototype`) or as an inline object (`customPrototype`) with `_biomekType: "technique"`.
- **Sources:** the four default projects (`Biomek_i3Project`, `Biomek_i5Project-MC`,
  `Biomek_i5Project-Span`, `Biomek_i7Project`) and the technique definitions each one
  installs into its Technique Browser.
- **Pod mapping used to derive `hardwareCompatibility`:** each pod tag comes from the
  single-pod project that owns that pod family — Fixed-8 from `Biomek_i3Project`,
  Multichannel from `Biomek_i5Project-MC`, Span-8 from `Biomek_i5Project-Span`. The
  `Biomek_i7Project` (Hybrid) project includes duplicates of the Multichannel and Span-8
  techniques but is not used to add tags.
- All default technique names are pod-family-prefixed and the corresponding
  `context.pod` list confines each technique to that pod family, so every entry here
  resolves to a single pod (`["Fixed-8"]`, `["Multichannel"]`, or `["Span-8"]`).

## Serialized JSON shape

Each technique object is emitted with:

- A `_biomekType: "technique"` discriminator at the top level.
- Nested plain-dictionary blocks — `aspirate`, `context`, `dispense`, `general`, `mix` — that
  carry no `_biomekType` (plain dictionaries are the "empty type" and elide the tag to keep
  the JSON concise, as documented in `Method-JSON-Structure.md`).
- Top-level scalars `name`, `readOnly`, and `template` (the read-only name of the pipetting
  template this technique binds to).
- Keys are camelCased on write and compared case-insensitively on read. `C__Blowout` in the
  source object surfaces as `c__Blowout` in JSON; `Aspirate` becomes `aspirate`; `Name`
  becomes `name`. Keys are sorted alphabetically (case-insensitive) within each block. Some
  default techniques carry fully lowercased key names (for example `minimumvolume` /
  `maximumvolume`) inherited from their source declarations while others use `minimumVolume`
  / `maximumVolume`; because keys are compared case-insensitively on read, the printed
  variation is cosmetic, not a typo.
- `context.*` fields carry the auto-select match criteria the technique is scoped to —
  `pod`, and any of `head`, `syringeType`, `tips`, `labware`, `minimumVolume`,
  `maximumVolume`, `rank`, `autoSelection`. Each is a JSON array (a variant list) even when it
  contains a single value.

The `customPrototype` payload embedded on a Multichannel Aspirate step in a real exported
method — see the JSON snippet on the Multichannel Aspirate step page — is a direct example
of this shape.

## Items

### Full technique payloads — one exemplar per pod family

These three entries are shown in full. Every other default technique in the section
below uses the same top-level shape as the exemplar for its pod family; the exact
`general`-block key set differs between Fixed-8 and Multichannel/Span-8 (see the note under
the `MC P60` example). The compact entries omit the `aspirate` / `dispense` / `mix` /
`general` blocks for space and list only the identifying `context`, `name`, `template`,
and `readOnly` fields — which capture the per-technique auto-select scope and template
pairing.

#### `F8 Medium` — Fixed-8 default pipetting technique

`hardwareCompatibility: ["Fixed-8"]`

```json
{
  "_biomekType": "technique",
  "aspirate": {
    "c__Blowout": true,
    "c__FollowLiquid": true,
    "c__Height": -2.0,
    "c__HeightFrom": 0,
    "c__Mix": false,
    "c__MixAspirateFrom": 0,
    "c__MixAspirateHeight": 0.0,
    "c__MixAspirateSpeed": 100.0,
    "c__MixCount": 1.0,
    "c__MixDispenseFrom": 0,
    "c__MixDispenseHeight": 0.0,
    "c__MixDispenseSpeed": 100.0,
    "c__MixVolume": 10.0,
    "c__Speed": 100.0,
    "c__TipTouch": true,
    "c__TrailingAirGap": false
  },
  "context": {
    "maximumVolume": [230.0],
    "minimumVolume": [10.0],
    "pod": ["Fixed8"]
  },
  "dispense": {
    "c__Blowout": true,
    "c__FollowLiquid": true,
    "c__Height": 0.0,
    "c__HeightFrom": 0,
    "c__Mix": false,
    "c__MixAspirateFrom": 0,
    "c__MixAspirateHeight": 0.0,
    "c__MixAspirateSpeed": 100.0,
    "c__MixCount": 1.0,
    "c__MixDispenseFrom": 0,
    "c__MixDispenseHeight": 0.0,
    "c__MixDispenseSpeed": 100.0,
    "c__MixVolume": 10.0,
    "c__Speed": 100.0,
    "c__TipTouch": false
  },
  "general": {
    "c__aspiratedelay": 0.0,
    "c__aspiratespeed": 50.0,
    "c__blowoutdelay": 0.0,
    "c__blowoutvolume": 15.0,
    "c__CalibrationOffset": 2.0,
    "c__CalibrationSlope": 1.03,
    "c__CDConductivity": "1",
    "c__CDConductivityInherit": 2,
    "c__CDErrorAutoRecover": 2,
    "c__CDHeight": "0",
    "c__CDHeightFrom": 0,
    "c__CDRepeat": "1",
    "c__CDSpeed": 0.0,
    "c__ConditioningExcess": 0.0,
    "c__ConditioningSectionCount": 0.0,
    "c__ConditioningSectionVolume": 0.0,
    "c__Conductivity": "3500",
    "c__ConductivityInherit": 2,
    "c__cutoffvelocity": 150.0,
    "c__DetectionSpeed": 20.0,
    "c__dispensedelay": 150.0,
    "c__dispensespeed": 25.0,
    "c__EmptyTips": false,
    "c__LLSAccept": false,
    "c__LLSAcceptValue": "0",
    "c__llserrorautorecover": 2,
    "c__llserrorautoretry": "0",
    "c__LLSFail": false,
    "c__LLSFailValue": "0",
    "c__LLSInitial": true,
    "c__LLSInitialHeight": "0",
    "c__LLSInitialHeightFrom": 2,
    "c__LLSRepeat": "1",
    "c__MinimumHeight": 0.2,
    "c__overrideliquidtypesettings": true,
    "c__PiercedSpeedLimit": 20.0,
    "c__PiercingExitHeight": 2.0,
    "c__PiercingExitSpeed": 20.0,
    "c__PiercingFinalHeight": -5.0,
    "c__PiercingFinalSpeed": 20.0,
    "c__PiercingInitialHeight": 2.0,
    "c__Prewet": false,
    "c__prewetdelay": 0.0,
    "c__prewetoverage": 0.0,
    "c__tiptouchdelay": 0.0,
    "c__tiptouchheight": -1.0,
    "c__tiptouchheightfrom": 2.0,
    "c__tiptouchspeed": 100.0,
    "c__tiptouchtheta": 90.0,
    "c__trailingairgapvolume": 0.0
  },
  "mix": {
    "c__Blowout": true,
    "c__FollowLiquid": false,
    "c__Mix": true,
    "c__MixAspirateFrom": 1,
    "c__MixAspirateHeight": 2.0,
    "c__MixAspirateSpeed": 100.0,
    "c__MixDispenseFrom": 1,
    "c__MixDispenseHeight": 2.0,
    "c__MixDispenseSpeed": 100.0,
    "c__Speed": 100.0,
    "c__TipTouch": true
  },
  "name": "F8 Medium",
  "readOnly": false,
  "template": "F8 Pipetting"
}
```

#### `MC P60` — Multichannel default, 384-head

`hardwareCompatibility: ["Multichannel"]`

```json
{
  "_biomekType": "technique",
  "aspirate": {
    "c__Blowout": true,
    "c__FollowLiquid": true,
    "c__Height": -2.0,
    "c__HeightFrom": 0,
    "c__Mix": false,
    "c__MixAspirateFrom": 1,
    "c__MixAspirateHeight": 0.0,
    "c__MixAspirateSpeed": 100.0,
    "c__MixCount": 1.0,
    "c__MixDispenseFrom": 1,
    "c__MixDispenseHeight": 0.0,
    "c__MixDispenseSpeed": 100.0,
    "c__MixVolume": 10.0,
    "c__Speed": 10.0,
    "c__TipTouch": false,
    "c__TrailingAirGap": false
  },
  "context": {
    "head": ["MC384_60"],
    "pod": ["Pod96"],
    "rank": [50]
  },
  "dispense": {
    "c__Blowout": true,
    "c__FollowLiquid": true,
    "c__Height": -2.0,
    "c__HeightFrom": 0,
    "c__Mix": false,
    "c__MixAspirateFrom": 1,
    "c__MixAspirateHeight": 2.0,
    "c__MixAspirateSpeed": 100.0,
    "c__MixCount": 4.0,
    "c__MixDispenseFrom": 1,
    "c__MixDispenseHeight": 5.0,
    "c__MixDispenseSpeed": 100.0,
    "c__MixVolume": 20.0,
    "c__Speed": 10.0,
    "c__TipTouch": true
  },
  "general": {
    "c__CalibrationOffset": 0.0,
    "c__CalibrationSlope": 1.0,
    "c__CDConductivity": "1",
    "c__CDConductivityInherit": 2,
    "c__CDErrorAutoRecover": 2,
    "c__CDHeight": "0",
    "c__CDHeightFrom": 0,
    "c__CDRepeat": "1",
    "c__CDSpeed": 30.0,
    "c__ConditioningExcess": 0.0,
    "c__ConditioningSectionCount": 0.0,
    "c__ConditioningSectionVolume": 0.0,
    "c__Conductivity": "2500",
    "c__ConductivityInherit": 0,
    "c__DetectionSpeed": 100.0,
    "c__EmptyTips": false,
    "c__LLSAccept": false,
    "c__LLSAcceptValue": "0",
    "c__llserrorautorecover": 2,
    "c__llserrorautoretry": "0",
    "c__LLSFail": false,
    "c__LLSFailValue": "0",
    "c__LLSInitial": true,
    "c__LLSInitialHeight": "0",
    "c__LLSInitialHeightFrom": 0,
    "c__LLSRepeat": "1",
    "c__MinimumHeight": 0.2,
    "c__overrideliquidtypesettings": false,
    "c__PiercedSpeedLimit": 20.0,
    "c__PiercingExitHeight": 2.0,
    "c__PiercingExitSpeed": 20.0,
    "c__PiercingFinalHeight": -5.0,
    "c__PiercingFinalSpeed": 20.0,
    "c__PiercingInitialHeight": 2.0,
    "c__Prewet": false
  },
  "mix": {
    "c__Blowout": true,
    "c__FollowLiquid": false,
    "c__Mix": true,
    "c__MixAspirateFrom": 1,
    "c__MixAspirateHeight": 3.0,
    "c__MixAspirateSpeed": 100.0,
    "c__MixDispenseFrom": 1,
    "c__MixDispenseHeight": 3.0,
    "c__MixDispenseSpeed": 100.0,
    "c__Speed": 10.0,
    "c__TipTouch": true
  },
  "name": "MC P60",
  "readOnly": false,
  "template": "MC Pipetting"
}
```

> Note: the Multichannel and Span-8 `general` blocks do not carry the free-standing
> `c__aspiratedelay`, `c__aspiratespeed`, `c__blowoutdelay`, `c__blowoutvolume`,
> `c__cutoffvelocity`, `c__dispensedelay`, `c__dispensespeed`, `c__prewetdelay`,
> `c__prewetoverage`, `c__tiptouchdelay`, `c__tiptouchheight`, `c__tiptouchheightfrom`,
> `c__tiptouchspeed`, `c__tiptouchtheta`, or `c__trailingairgapvolume` keys that Fixed-8
> techniques do — those parameters are supplied by liquid-type context when
> `c__overrideliquidtypesettings` is `false`.

#### `S8 1000 Medium` — Span-8 default, 1000 syringes, mid volume

`hardwareCompatibility: ["Span-8"]`

```json
{
  "_biomekType": "technique",
  "aspirate": {
    "c__Blowout": true,
    "c__FollowLiquid": true,
    "c__Height": -2.0,
    "c__HeightFrom": 0,
    "c__Mix": false,
    "c__MixAspirateFrom": 0,
    "c__MixAspirateHeight": 0.0,
    "c__MixAspirateSpeed": 100.0,
    "c__MixCount": 1.0,
    "c__MixDispenseFrom": 0,
    "c__MixDispenseHeight": 0.0,
    "c__MixDispenseSpeed": 100.0,
    "c__MixVolume": 10.0,
    "c__Speed": 5.0,
    "c__TipTouch": false,
    "c__TrailingAirGap": true
  },
  "context": {
    "minimumVolume": [25.0],
    "pod": ["Pod8Span"],
    "syringeType": ["SYR1000", "SYR2500", "SYR5000"]
  },
  "dispense": {
    "c__Blowout": true,
    "c__FollowLiquid": false,
    "c__Height": 1.5,
    "c__HeightFrom": 1,
    "c__Mix": false,
    "c__MixAspirateFrom": 0,
    "c__MixAspirateHeight": 0.0,
    "c__MixAspirateSpeed": 100.0,
    "c__MixCount": 1.0,
    "c__MixDispenseFrom": 0,
    "c__MixDispenseHeight": 0.0,
    "c__MixDispenseSpeed": 100.0,
    "c__MixVolume": 10.0,
    "c__Speed": 5.0,
    "c__TipTouch": false
  },
  "general": {
    "c__CalibrationOffset": 0.0,
    "c__CalibrationSlope": 1.0,
    "c__CDConductivity": "1",
    "c__CDConductivityInherit": 2,
    "c__CDErrorAutoRecover": 2,
    "c__CDHeight": "0",
    "c__CDHeightFrom": 0,
    "c__CDRepeat": "1",
    "c__CDSpeed": 30.0,
    "c__ConditioningExcess": 0.0,
    "c__ConditioningSectionCount": 0.0,
    "c__ConditioningSectionVolume": 0.0,
    "c__Conductivity": "3500",
    "c__ConductivityInherit": 2,
    "c__DetectionSpeed": 20.0,
    "c__EmptyTips": false,
    "c__LLSAccept": false,
    "c__LLSAcceptValue": "0",
    "c__llserrorautorecover": 2,
    "c__llserrorautoretry": "1",
    "c__LLSFail": false,
    "c__LLSFailValue": "0",
    "c__LLSInitial": true,
    "c__LLSInitialHeight": "0",
    "c__LLSInitialHeightFrom": 2,
    "c__LLSRepeat": "1",
    "c__MinimumHeight": 1.5,
    "c__overrideliquidtypesettings": false,
    "c__PiercedSpeedLimit": 20.0,
    "c__PiercingExitHeight": 2.0,
    "c__PiercingExitSpeed": 20.0,
    "c__PiercingFinalHeight": -5.0,
    "c__PiercingFinalSpeed": 20.0,
    "c__PiercingInitialHeight": 2.0,
    "c__Prewet": false
  },
  "mix": {
    "c__Blowout": true,
    "c__FollowLiquid": false,
    "c__Mix": true,
    "c__MixAspirateFrom": 1,
    "c__MixAspirateHeight": 3.0,
    "c__MixAspirateSpeed": 100.0,
    "c__MixDispenseFrom": 1,
    "c__MixDispenseHeight": 3.0,
    "c__MixDispenseSpeed": 100.0,
    "c__Speed": 5.0,
    "c__TipTouch": false
  },
  "name": "S8 1000 Medium",
  "readOnly": false,
  "template": "S8 Pipetting"
}
```

### Compact entries — remaining defaults

Each entry below shows the identifying top-level fields (`_biomekType`, `context`, `name`,
`readOnly`, `template`) — the auto-select scope and template binding. The `aspirate`,
`dispense`, `mix`, and `general` blocks are omitted for space; they follow the same key
set as the pod-family exemplar above and carry values tuned to the technique (open the
technique in the Technique Editor to see the resolved values on the General / Aspirate /
Dispense / Mix / Calibration / Liquid Level Sensing / Clot Detection / Piercing tabs).

#### Fixed-8 techniques (`hardwareCompatibility: ["Fixed-8"]`)

All Fixed-8 techniques are included in `Biomek_i3Project` only.

##### `F8 High`

```json
{
  "_biomekType": "technique",
  "context": {
    "minimumVolume": [100.0],
    "pod": ["Fixed8"]
  },
  "name": "F8 High",
  "readOnly": false,
  "template": "F8 Pipetting"
}
```

##### `F8 Low`

```json
{
  "_biomekType": "technique",
  "context": {
    "maximumVolume": [90.0],
    "pod": ["Fixed8"]
  },
  "name": "F8 Low",
  "readOnly": false,
  "template": "F8 Pipetting"
}
```

##### `F8 Medium TipDipDispense`

```json
{
  "_biomekType": "technique",
  "context": {
    "maximumVolume": [230.0],
    "minimumVolume": [10.0],
    "pod": ["Fixed8"]
  },
  "name": "F8 Medium TipDipDispense",
  "readOnly": false,
  "template": "F8 Pipetting TipDipDispense"
}
```

##### `F8 Medium TopDispense`

```json
{
  "_biomekType": "technique",
  "context": {
    "maximumVolume": [230.0],
    "minimumVolume": [10.0],
    "pod": ["Fixed8"]
  },
  "name": "F8 Medium TopDispense",
  "readOnly": false,
  "template": "F8 Pipetting"
}
```

##### `F8 MultiDispense`

```json
{
  "_biomekType": "technique",
  "context": {
    "autoSelection": ["DoNotAutoSelect"],
    "pod": ["Fixed8"],
    "rank": [50]
  },
  "name": "F8 MultiDispense",
  "readOnly": false,
  "template": "F8 Pipetting"
}
```

> Fixed-8 pipetting steps (Aspirate/Dispense/Mix) never auto-select a technique — the step
> code throws if `autoSelectPrototype` is `true`, and otherwise requires an explicit
> `prototype` (or `customPrototype`). This is enforced in the step itself, independent of
> any technique's `DoNotAutoSelect`/auto-selection flag (those only affect the shared
> auto-select resolver used by the other pod families).

#### Multichannel techniques (`hardwareCompatibility: ["Multichannel"]`)

All Multichannel techniques are included in `Biomek_i5Project-MC` and `Biomek_i7Project`.

##### `MC`

```json
{
  "_biomekType": "technique",
  "context": {
    "head": ["MC96_1200", "MC96_300"],
    "pod": ["Pod96"],
    "rank": [50]
  },
  "name": "MC",
  "readOnly": false,
  "template": "MC Pipetting"
}
```

##### `MC Active Wash`

```json
{
  "_biomekType": "technique",
  "context": {
    "labware": ["WashStation", "WashStation384"],
    "pod": ["Pod96"],
    "rank": [25]
  },
  "name": "MC Active Wash",
  "readOnly": false,
  "template": "MC Active Washing"
}
```

##### `MC Low`

```json
{
  "_biomekType": "technique",
  "context": {
    "head": ["MC96_1200", "MC96_300"],
    "maximumVolume": [5.0],
    "pod": ["Pod96"],
    "rank": [25]
  },
  "name": "MC Low",
  "readOnly": false,
  "template": "MC Low-Volume Pipetting"
}
```

##### `MC MultiDispense`

```json
{
  "_biomekType": "technique",
  "context": {
    "autoSelection": ["DoNotAutoSelect"],
    "pod": ["Pod96"],
    "rank": [50]
  },
  "name": "MC MultiDispense",
  "readOnly": false,
  "template": "MC Pipetting"
}
```

##### `MC P60 Low`

```json
{
  "_biomekType": "technique",
  "context": {
    "head": ["MC384_60"],
    "maximumVolume": [5.0],
    "pod": ["Pod96"],
    "rank": [25]
  },
  "name": "MC P60 Low",
  "readOnly": false,
  "template": "MC Low-Volume Pipetting"
}
```

##### `MC P300 High`

```json
{
  "_biomekType": "technique",
  "context": {
    "head": ["MC96_300"],
    "minimumvolume": [50.0],
    "pod": ["Pod96"],
    "rank": [25]
  },
  "name": "MC P300 High",
  "readOnly": false,
  "template": "MC Pipetting"
}
```

##### `MC P1200 High`

```json
{
  "_biomekType": "technique",
  "context": {
    "head": ["MC96_1200"],
    "minimumvolume": [150.0],
    "pod": ["Pod96"],
    "rank": [25]
  },
  "name": "MC P1200 High",
  "readOnly": false,
  "template": "MC Pipetting"
}
```

> The Multichannel head identifiers describe which multichannel head is installed on the
> pod: `MC96_1200` and `MC96_300` are the 96-format 1200 uL and 300 uL heads; `MC384_60`
> is the 384-format 60 uL head. Auto-select uses `head` to scope which volume-range
> techniques apply on a given assembly.

#### Span-8 techniques (`hardwareCompatibility: ["Span-8"]`)

All Span-8 techniques are included in `Biomek_i5Project-Span` and `Biomek_i7Project`.

##### `S8 250`

```json
{
  "_biomekType": "technique",
  "context": {
    "minimumVolume": [5.0],
    "pod": ["Pod8Span"],
    "syringeType": ["SYR250", "SYR500"]
  },
  "name": "S8 250",
  "readOnly": false,
  "template": "S8 Pipetting"
}
```

##### `S8 250 Low`

```json
{
  "_biomekType": "technique",
  "context": {
    "maximumvolume": [5.0],
    "pod": ["Pod8Span"],
    "syringeType": ["SYR250", "SYR500"]
  },
  "name": "S8 250 Low",
  "readOnly": false,
  "template": "S8 Pipetting"
}
```

##### `S8 1000 Low`

```json
{
  "_biomekType": "technique",
  "context": {
    "maximumvolume": [25.0],
    "pod": ["Pod8Span"],
    "syringeType": ["SYR1000", "SYR2500", "SYR5000"]
  },
  "name": "S8 1000 Low",
  "readOnly": false,
  "template": "S8 Pipetting"
}
```

##### `S8 1000 High`

```json
{
  "_biomekType": "technique",
  "context": {
    "minimumVolume": [500.0],
    "pod": ["Pod8Span"],
    "syringeType": ["SYR1000", "SYR2500", "SYR5000"]
  },
  "name": "S8 1000 High",
  "readOnly": false,
  "template": "S8 Pipetting"
}
```

##### `S8 Active Wash`

```json
{
  "_biomekType": "technique",
  "context": {
    "labware": ["WashStation", "WashStation384", "WashStationSpan8"],
    "pod": ["Pod8Span"]
  },
  "name": "S8 Active Wash",
  "readOnly": false,
  "template": "S8 Active Washing"
}
```

##### `S8 Clot Detection`

```json
{
  "_biomekType": "technique",
  "context": {
    "autoSelection": ["DoNotAutoSelect"],
    "pod": ["Pod8Span"],
    "tips": [
      "Fixed100",
      "SeptaFluted",
      "T1025F_LLS",
      "T1070_LLS",
      "T190F_LLS",
      "T230_LLS",
      "T40F_LLS",
      "T50F_LLS",
      "T80_LLS",
      "T90_LLS"
    ]
  },
  "name": "S8 Clot Detection",
  "readOnly": false,
  "template": "S8 Clot Detection"
}
```

##### `S8 MultiDispense`

```json
{
  "_biomekType": "technique",
  "context": {
    "autoSelection": ["DoNotAutoSelect"],
    "pod": ["Pod8Span"]
  },
  "name": "S8 MultiDispense",
  "readOnly": false,
  "template": "S8 Pipetting"
}
```

##### `S8 SeptaFluted`

```json
{
  "_biomekType": "technique",
  "context": {
    "labware": [
      "BCSeptaTubeRack_13x100mm",
      "BCSeptaTubeRack_13x75mm",
      "BCSeptaTubeRack_15_5x100mm",
      "BCSeptaTubeRack_15_5x75mm"
    ],
    "pod": ["Pod8Span"],
    "tips": ["SeptaFluted"]
  },
  "name": "S8 SeptaFluted",
  "readOnly": false,
  "template": "S8 Septa Piercing"
}
```

> Span-8 syringe identifiers (`SYR250`, `SYR500`, `SYR1000`, `SYR2500`, `SYR5000`) name
> the installed syringe size on the Span-8 pod's 8 channels. Auto-select uses
> `syringeType` to scope which pipetting-volume techniques apply on a given assembly.
