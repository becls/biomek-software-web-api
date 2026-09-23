# ALP Catalog

This catalog covers the common default ALPs (Automated Labware Positioners) whose
`alpType` values can appear in a Biomek deck layout — including
the ALPs that no sample deck in `samples/` exercises. When you encounter an
`alpType` in a `deckLayouts` JSON export, this is a good starting reference for
what that ALP is and which instruments it is typically used on.

**This list is not exhaustive.** Additional ALP types beyond those listed here
may appear on a user's system — for example, ALPs installed as part of add-on
hardware kits or integrated third-party devices. Treat this catalog as a
description of the common defaults, not a closed enumeration of every `alpType`
you might encounter.

For the JSON field layout that surrounds `alpType`, see
[Deck Layouts Format Specification](./deck-layouts-format-spec.md). For quick visual
examples of the default decks included with Biomek, see [samples/](./samples/).

## Hardware-compatibility conventions used below

- **Biomek i3** — compact chassis with a Fixed-8 pod. ALPs whose `alpType` ends in
  `_i3` are the i3-chassis variants and are only used on i3 decks.
- **i5 and i7** — Each i5/i7 instrument carries either
  a Multichannel pod (96-tip or 384-tip head) or a Span-8 pod, or both (Hybrid /
  Dual-Multichannel).
- "Any pod" means the ALP is a passive holder or device platform that can be
  reached by whichever pod the instrument is equipped with; a pod-specific note calls
  out ALPs (wash stations, tube racks) whose labware or geometry is tied to one
  pod family.

The compatibility column below reflects each ALP's **default
applicability** as expressed by its category tags and its type-name convention;
it is not an absolute mechanical constraint enforced by the instrument.

## Common ALP types

| `alpType` | What it is | Hardware compatibility |
|---|---|---|
| `Static1x1` | Passive holder that presents one titer-plate-sized position. | i5 and i7; any pod. |
| `Static1x3` | Passive holder with three titer-plate positions stacked front-to-back on the same footprint. | i5 and i7; any pod. |
| `Static1x5` | Passive holder with five titer-plate positions stacked front-to-back — the largest tandem static holder. | i5 and i7; any pod. |
| `Static1x1_i3` | Single-plate passive holder sized and mounted for the compact i3 deck grid. | Biomek i3 chassis (Fixed-8 pod). |
| `Short1x1_i3` | Low-profile single-plate holder for the i3 — reduced Z so the Fixed-8 pod can reach plate labware inside its limited Z-travel range. Used to hold tall tip boxes (such as BC1070). | Biomek i3 chassis (Fixed-8 pod). |
| `TipLoad1x1` | Single-plate holder designed for the Multichannel pod to load tips from. Multichannel pods cannot load tips from any other ALP type. Can also be used for pipetting or Span-8 tip loading. | i5 and i7; supports tip loading with both Multichannel and Span-8 pods. |
| `TipTrash_i3` | Discarded-tip trash for the i3, feeding a bag or bin. Not a labware position — accepts no plates and cannot be framed. | Biomek i3 chassis (Fixed-8 pod). |
| `WashStation96` | Circulating wash reservoir sized for a 96-tip Multichannel pod. | i5 or i7 with a 96-tip Multichannel pod. |
| `WashStation384` | Circulating wash reservoir sized for a 384-tip Multichannel pod. | i5 or i7 with a 384-tip Multichannel pod. |
| `WashStationSpan8` | Dispense-only wash reservoir with eight wash points for the Span-8 probe pod. Narrow tower footprint mounted off to the side of the deck. | i5 or i7 with a Span-8 pod. |
| `WashStationSpan8Active` | Circulating wash reservoir for the Span-8 pod. Same envelope as the passive Span-8 wash station. | i5 or i7 with a Span-8 pod. |
| `TrashLeftBin` | Bin-style trash chute mounted on the left side of the deck. Accepts discarded tips and labware. | i5 and i7; default trash accessory (categorized with Span-8 in the Deck Editor). |
| `TrashLeftSlide` | Slide-style trash chute mounted on the left side of the deck — same function as `TrashLeftBin` with a slide form-factor. | i5 and i7; default trash accessory. |
| `TrashRightBin` | Bin-style trash chute mounted on the right side of the deck (mirror of `TrashLeftBin`). | i5 and i7; default trash accessory. |
| `TrashRightSlide` | Slide-style trash chute mounted on the right side of the deck (mirror of `TrashLeftSlide`). | i5 and i7; default trash accessory. |
| `TubeRack` | Holder for a tube-rack labware — the entry point for single-tube pipetting with the Span-8 probes. | i5 or i7 with a Span-8 pod. |
| `OrbitalShaker` | Mounting platform for an orbital shaker device that agitates a plate in place. | i5 and i7; any pod (device platform). |
| `HeatOrCool` | Mounting platform for a Peltier-based heat/cool device that regulates a plate's temperature. | i5 and i7; any pod (device platform). Also qualified for Span-8 tip pickup. |
| `ReservoirTipBox` | Mounting platform for a circulating-reservoir labware that can hold a tip box on top of it (for on-deck bulk-liquid supply). | i5 and i7; any pod (device platform). |
| `PositivePositioner` | Mounting platform for a clamp device that mechanically pushes a plate against known X/Y stops so its origin is exact. | i5 and i7; any pod (device platform). |
| `FBBCR` | Mounting platform for a bar-code reader that scans plate labels. Presents a scan-target position, not a labware seat. | i5 and i7; any pod (device platform). |

## Notes

- **Multi-position ALPs** (`Static1x3`, `Static1x5`) contribute more than one
  usable position on the deck; each member position has its own key in the JSON
  export. See the format spec's group / position sections for how the members
  are laid out.
- **Permanent-labware ALPs** — the active wash stations — come with a fixed piece of
  labware pre-installed at the position; method authors don't place labware on
  them via Instrument Setup.
- **Read-only trash devices** — the four `Trash*` ALPs (and `TipTrash_i3`) carry
  factory-defined trash devices that are fixed to the deck; they are part of the deck
  geometry, not something a method configures or reassigns.
