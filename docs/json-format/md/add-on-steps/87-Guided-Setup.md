# Guided Setup

| Property | Value |
|----------|-------|
| stepType | **`"Guided Labware Setup"`** |
| Category | Leaf (SubStepsReadOnly) |
| Terminator | N/A |
| Compatible Hardware | All (requires the Guided Labware Setup add-on installed and registered) |

> Base keys + casing: see [Base Keys & Serialization Conventions](../step-documentation/00-Base-Keys-and-Conventions.md).

> **Add-on prerequisite.** Guided Setup is an add-on step rather than a built-in, and
> lives in **`Biomek5GuidedLabwareLib.dll`**. The target Biomek install **must have this
> library installed and registered** for import to succeed — on a stock Biomek without the
> library, authoring this step fails at import as an unresolved-CLSID error. If the target
> lacks the library, use plain Instrument Setup (see [Instrument Setup](../step-documentation/03-Instrument-Setup.md)) instead.
>

The minimal authored form:

```json
{
  "stepType": "Guided Labware Setup",
  "parameters": {
    "deck": "i7 Standard"
  }
}
```

## Behavior Summary

The Guided Setup step configures the instrument deck at the start of a method run. It verifies pod configuration, loads a named deck layout — preserving labware at any "AsIs" positions and clearing the remaining ones — and runs custom setup scripts. At the end it presents the operator with a guided labware placement dialog, pausing the run for confirmation.

It is an ordinary registered step — any Biomek install that has the add-on registered can run it, in any method. In the editor it appears on the "Setup & Device Steps" palette. Along the way it also handles labware group management, barcode scanning, deck light control, pod parking, and light curtain/interlock coordination.

## Parameters Reference Table

### Core Parameters

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `deck` | `string` | — | Yes | — | Deck layout name. Must name a deck layout defined on the instrument. |
| `groups` | `array` | — | No | — | Array of labware group configuration objects. Each group has `name`, `labwareList`, and `notes`. |
| `asIsPositions` | `array` | — | No | `[]` *(empty list)* | List of deck position names whose existing labware should be preserved during layout loading. |
| `initialKey` | `string` | — | No | `""` | Setup notes text shown to the operator before confirmation. Supports expression evaluation. |
| `permanentLabware` | `object (dictionary)` | — | No | `{}` *(empty dictionary)* | Keyed by deck position name and contains a subdictionary with only the LiquidType key and value associated with the permanent labware at that position. Populated at design time from deck positions that carry the instrument's own permanent-labware flag; the LiquidType may be modified, but no other parameters should be added to the subdictionary for each position. The permanent labware associated with the deck position will be used with the correct liquid type as provided upon import.

### Script Parameters

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `code` | `string` | — | No | `""` | Custom script code to execute during setup (before labware placement). |
| `isVBScript` | `boolean` | — | No | `false` | If `true`, `code` is VBScript; if `false`, JavaScript. |

### Confirmation & Behavior Parameters

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `confirmSetup?` | `boolean` | — | No | `true` | If `true`, pause for operator to confirm deck setup. |
| `verifyPodSetup?` | `boolean` | — | No | `true` | If `true`, verify pod configuration before proceeding. |
| `turnDeckLightOn?` | `boolean` | — | No | `true` | If `true`, turn on deck light during setup. |

### Pod Setup Parameters (nested inside `podSetup` dictionary)

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `podSetup` | `object` | — | No | — | Dictionary containing pod configuration to verify. |
| `podSetup.leftHasTips` | `boolean` | — | No | — | Left pod should have tips loaded. |
| `podSetup.rightHasTips` | `boolean` | — | No | — | Right pod should have tips loaded. |
| `podSetup.leftTipType` | `string` | — | No | — | Expected left tip type. |
| `podSetup.rightTipType` | `string` | — | No | — | Expected right tip type. |
| `podSetup.washLiquidType` | `string` | — | No | — | Recognized key name, but **not read by pod verification**. Do not author it; preserve it verbatim if an existing method already carries it. |

> **Scope of `podSetup` verification.** Runtime pod verification reads exactly four keys:
> `leftHasTips`, `rightHasTips`, `leftTipType`, `rightTipType`. For each pod it brings the modeled tip
> state in line with those keys and appends a sentence describing the result to the setup
> dialog text the operator sees. Independently of any key, it also releases anything the pod's
> gripper is modeled as holding and, if it was holding something, adds "Ensure the … pod is not
> holding any labware." to that same text.
>
> Gripper- and tool-verification key names — `checkLeftGripper`, `checkRightGripper`,
> `leftHasTool`, `leftToolType`, `rightHasTool`, `rightToolType`, `toolSourceLoc`,
> `toolSourceSlot` — exist as recognized names but are **not** part of runtime pod verification.
> **Do not author them.**
>

### UI State Parameters (design-time only)

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `customColors` | `variant list` | — | No | *(empty list)* | UI color configuration. Editor-written; not read at runtime. Written on every editor save — see the note below the table. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, `isPreconfigured`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](../step-documentation/00-Base-Keys-and-Conventions.md). This step can carry `caption` and `isPreconfigured`; both are editor state — preserve them verbatim if present, never invent them._

> **Note — why these keys are Optional but always present in editor-produced steps.** Every parameter documented
> above is written **unconditionally** by the step's editor panel each time the step is saved:
> `deck`, `groups`, `asIsPositions`, `initialKey`, `permanentLabware`, `code`, `isVBScript`,
> `confirmSetup?`, `verifyPodSetup?`, `turnDeckLightOn?`, `podSetup`, `customColors`.
> So a step authored in the editor carries the whole set.
>
> That is a property of the editor, not a requirement of the run. Enqueue reads every one of
> them through a defaulting accessor or an `IsBound` guard, so the step still enqueues with any
> of them absent — `turnDeckLightOn?`, for instance, falls back to `true`. Keep them all
> Optional, and preserve exactly what an existing method carries when round-tripping — do not
> add a key a method does not already have.

## Labware Group Structure

Each element in the `groups` array is a dictionary with:

| Key | Type | Description |
|-----|------|-------------|
| `name` | `string` | Group name (must be unique across all groups) |
| `labwareList` | `array` | Labware items in this group. Emit `[]` when authoring a new group — see below. |
| `notes` | `string` | Operator notes for this group |

### Labware items — do not hand-author

Each element of `labwareList` is a **fully serialized labware object**, not a small descriptor. In
real methods an item carries the labware class reference, position, volume type, per-well amount and
liquid-type arrays (96+ entries each), a `properties` sub-dictionary (name, barcode, liquid type,
device, sense flag), and a set of internal markers that Guided Setup writes itself:

| Marker | Role |
|--------|------|
| `_is_brt_lwindex` | Internal identity of the item within its group |
| `position` | Target deck position (may be an expression) |
| `_is_brt_stackdepth` | Stack depth at that position; `1` is the bottom item |
| `use` | Whether the item is used on this run (may be an expression) |
| `wellNotesLine1` | Per-well label text shown in the placement dialog |
| `wellColors` | Per-well RGB values sent to the display |
| `notes` | Operator notes for this item |
| `hasBarcode` | Pause for a barcode read on this item |
| `advancedSetup` | Item still requires operator setup |
| `_is_brt_valid_pos` | Written by the step: whether the resolved position exists on the deck |
| `wellNotesLine2` | Per-well "line 2" label text shown in the placement dialog |
| `_is_brt_colorsource` | Recognized marker name with no reader or writer in the current step; preserve verbatim if present |

**Do not author `labwareList` entries or any of the above markers by hand** — populated groups are
produced by the Guided Setup editor, and a partially-formed item will not round-trip. Emit
`"labwareList": []` for a new group, and when round-tripping an existing method, preserve every
labware item and every marker verbatim.

## Cross-Field Validation Rules

1. `deck` **must not** be empty (fails as a no-deck-selected error).
2. `deck` **must** name a deck layout defined on the instrument (fails as a deck-not-defined error).
3. All group `name` values **must be unique** (fails as a duplicate-group-name error).
4. If `confirmSetup?` is `false` and any labware requires barcode input, the step fails as a
barcode-requires-confirmation error. Note that this includes labware with the TubeScanRack
characteristic even if hasBarcode is false. This is **not** a design-time guard: it is raised late
in the enqueue sequence, after the deck has been loaded, groups processed, and setup scripts run —
so earlier side effects are already queued when the error surfaces. Author `confirmSetup?: true`
whenever any labware carries barcode input.

## Enqueue Sequence

1. **Setup notes** — Evaluates `initialKey` as an expression and seeds the operator message with the result.
2. **Pod speed** — Sets Pod1 and Pod2 speeds to 100 (if present).
3. **Pod setup verification** — If `verifyPodSetup?` is `true` and `podSetup` is bound, brings the modeled pod state in line with the four tip keys and appends a description to the operator message.
4. **Deck validation and deck-name warning** — Rejects an empty or undefined `deck`; then, if the currently-loaded deck has a different name, appends a warning to the operator message. This happens **before** any deck loading, group processing, or script execution.
5. **Deck loading** — Loads the named deck layout and removes labware from positions that are neither in `asIsPositions` nor flagged permanent on the deck position itself.
6. **Labware group processing** — Creates labware groups from `groups`, validates unique names, fills labware dictionaries.
7. **Script execution** — Runs custom `code` script, then any plugin scripts installed at `%ProgramData%\Beckman Coulter\Guided Labware Setup\Scripts\*.vbs`.
8. **Barcode/confirmation check** — If `confirmSetup?` is `false` and any labware requires barcode input, raises the error described under Cross-Field Validation.
9. **Deck light** — Sets the deck light to `turnDeckLightOn?`. Skipped entirely when simulating.
10. **Pod parking** — Parks Pod1 and Pod2 at their park locations. **Skipped when simulating or when `confirmSetup?` is `false`.**
11. **Guided action** — Enqueues the interactive labware-placement dialog (pauses the light curtain/interlock, presents the UI, resumes). **Skipped when simulating or when `confirmSetup?` is `false`.**

In simulation the step still evaluates notes, loads the deck, processes groups, and runs scripts — it
just does not touch the deck light, does not park pods, and does not enqueue the operator dialog.

## Structural Context

This step is a leaf (SubStepsReadOnly — no user-editable sub-steps).

## Canonical Examples

Basic guided setup:

```json
{
  "stepType": "Guided Labware Setup",
  "parameters": {
    "deck": "i7 Standard",
    "confirmSetup?": true,
    "verifyPodSetup?": true,
    "turnDeckLightOn?": true
  }
}
```

Guided setup with groups and AsIs positions:

```json
{
  "stepType": "Guided Labware Setup",
  "parameters": {
    "deck": "i5 Span-8",
    "asIsPositions": ["P1", "P2"],
    "groups": [
      {
        "name": "Source Plates",
        "labwareList": [],
        "notes": "Place source plates in positions P3-P5"
      },
      {
        "name": "Destination Plates",
        "labwareList": [],
        "notes": "Place destination plates in positions P6-P8"
      }
    ],
    "initialKey": "Ensure tip boxes are full before starting.",
    "confirmSetup?": true,
    "verifyPodSetup?": false,
    "turnDeckLightOn?": true
  }
}
```

## Common Mistakes

- Disabling confirmation with barcode input: if any labware needs barcode scanning, `confirmSetup?` must be `true`. Enqueue rejects the combination outright, and does so *after* the deck load and setup scripts have already been queued.
- Fabricating `podSetup.*` gripper/tool keys: pod verification acts on four keys only — `leftHasTips`, `rightHasTips`, `leftTipType`, `rightTipType`. `checkLeftGripper`, `leftToolType`, `toolSourceLoc` and the rest are recognized names but are not part of verification; `washLiquidType` is likewise inert. Do not author any of them.
- **Hand-authoring a populated `groups` entry**: labware items are full serialized labware objects with internal markers. Emit `"labwareList": []` and let the editor populate it; preserve existing items verbatim.
- **Assuming the guided dialog always runs**: pod parking and the operator dialog are skipped in simulation and whenever `confirmSetup?` is `false`, even though the deck load and scripts still execute.
