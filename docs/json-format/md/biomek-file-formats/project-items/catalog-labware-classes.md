# Catalog: Labware Classes

> **The names here are real and usable.** These entries are the **default project items included
> with Biomek**: a method references them **by name**, and these are the valid names to use. Project
> items are not a file you author or import. A specific project may add its own, so treat this as
> the default set, not an exhaustive list.
>
> Every unique Beckman default labware class is listed once, deduplicated across the three
> single-pod default projects. The **Pods** column in each table records `hardwareCompatibility`,
> which is **pod-based** (Multichannel / Span-8 / Fixed-8); chassis size does not affect it, and a
> hybrid instrument supports the union of its pods. The field is tentative: it is a catalog annotation
> derived from which default projects include the item (mapped to their pods), not necessarily an
> intrinsic field of the forthcoming format. A Pods value of `any` means all three default projects
> include the class.
>
> **The Pods column is a membership hint, not a guarantee that the pod can use the labware.** It
> says the class is included in that pod's default project; it does not promise the pod can reach the
> position you place it at, or that the pod can access that labware type at all. A class marked
> `any` can still fail with `Cannot access labware of type <class> with <pod>.`, and a class marked
> for one pod family can still be unreachable where you put it. Reach is a property of the deck
> position and the pod — see
> [Reachability and Access](../deck-layouts/reachability-and-access.md).

## Scope and derivation

- **Category:** Labware Classes (`Labware Classes`).
- **Sources:** the three single-pod default projects (`Biomek_i3Project`,
  `Biomek_i5Project-MC`, `Biomek_i5Project-Span`).
- **Pod mapping used to derive `hardwareCompatibility`:** each pod tag is derived from the
  single-pod default project that uses that pod (`Fixed-8` from `Biomek_i3Project`,
  `Multichannel` from `Biomek_i5Project-MC`, `Span-8` from `Biomek_i5Project-Span`); an item's
  compatibility is the set of those pods whose project includes it, or `["any"]` when all three
  do. The `Biomek_i7Project` (Hybrid) is not used to add pod tags — it carries both pods, so
  membership there cannot distinguish which pod an item is for. Excluding it costs no coverage:
  its labware classes are exactly the union of the three single-pod projects.

This catalog covers six item types: reservoirs, titer plates, tip boxes, tube racks, lids, and
reservations. Reservoirs come first and get the most space because their addressing model differs
from every other type — a reservoir is a list of sections, not a well grid.

## Reservoirs

A reservoir labware class is **not** a WellsX-by-WellsY grid. It is modeled as a **`Sections[]`
array**, and each section is the addressable unit for pipetting. A section is addressed by its
**1-based position** in the type's section list. When authoring a pipetting step that targets a
reservoir:

- The `selectionInfo` / `pattern` boolean comArray length equals the reservoir's **section count**
  (one flag per section position).
- Each flag selects (`true`) or deselects (`false`) the section at that position.
- Section ordering is **author-defined** in the Labware Type Editor's Reservoir Builder
  (sorting mode + manual Section ID), so section 1 is not necessarily the leftmost cavity.

For the full numbering rule see
[Reservoirs and multi-section troughs](../../Introduction-to-Pipetting-Steps.md#reservoirs-and-multi-section-troughs)
in Introduction-to-Pipetting-Steps.md. For deck placement see the
[sample deck layouts](../deck-layouts/samples/).

### Quick reference

| Class | Sections | Max volume per section (µL) | Refills | Top-section depth (cm) | Bottom shape | Pods | Description |
|---|---|---|---|---|---|---|---|
| `AgilentReservoir` | 1 | 300000 | No | 3.922 | Flat | any | Agilent Seahorse Microplates (formerly Innovative Microplate): single cavity polypropylene reservoir, 300ml, 96 pyramids base geometry, 44mm height. |
| `BCFullReservoir` | 1 | 200000 | No | 2.159 | V-shaped | Span-8 | Beckman Coulter reservoir with baffles used to hold bulk reagents for dispensing by the Span-8 Pod |
| `BCUpsideDownTipBoxLid` | 1 | 160000 | No | 1.71 | Flat | any | Upside down lid from a Beckman Coulter tip box used as a reservoir |
| `CirculatingReservoir` | 1 | 250000 | Yes | 3.78 | Flat | Multichannel, Span-8 | Circulating reservoir for use with the ReservoirTipBox ALP |
| `DrainableRefillableReservoir` | 1 | 71606 | No | 1.905 | Flat | Multichannel, Span-8 | Beckman Coulter drainable and refillable reservoir |
| `ModularReservoir` | 4 | 40700 | No | 2.479 | V-shaped | any | Beckman Coulter reservoir that may be customized with added sections |
| `Reservoir` | 1 | 110000 | No | 1.453 | Flat | any | Reservoir used with the Multimek |

All seven carry `Can Pipette` and `Standard TiterPlate Size`. Only `AgilentReservoir`,
`BCFullReservoir`, and `Reservoir` also carry `Moveable`; a Move Labware step cannot reposition
the other four. **Refills** marks a reservoir replenished from an external source during the run,
which among the defaults is only `CirculatingReservoir`. **Top-section depth** is the Z-span of the
section's upper, straight-walled portion; each class has a further bottom section beneath it, so
this figure is not the full distance to the cavity floor. `ModularReservoir`'s default
configuration is four equal quarter-sections arranged left to right.

### Worked example: Transfer step items targeting a reservoir

Aspirating from a single-section reservoir (`BCFullReservoir`) — the `pattern` and
`selectionInfo` arrays each have **1** element (one per section):

```jsonc
// Source item inside a Transfer step's "items" array
{
  "labwareClass": "BCFullReservoir",
  "position": "P3",
  "liquidType": "Well Contents",
  "volume": "200",
  // ... other Transfer-item keys ...
  "pattern": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true] },
  "selectionInfo": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true] }
}
```

For a 4-section `ModularReservoir`, select only section 2 by setting the second flag:

```jsonc
{
  "labwareClass": "ModularReservoir",
  "position": "P3",
  "liquidType": "Well Contents",
  "volume": "100",
  // pattern length = 4 (one per section):
  "pattern": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [false, true, false, false] },
  "selectionInfo": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [false, true, false, false] }
}
```

**Best practice: aspirate from reservoirs bottom-relative, not liquid-relative.** Reservoir
aspiration is conventionally referenced to the section bottom (`overrideHeight: true`,
`heightFrom: 1` = `"Bottom"`, with a small positive `height` above the floor). Fixed-8 and
Multichannel pods do not perform liquid-level sensing (only Span-8 does — see the LLS note
in [Catalog: Tip Classes](catalog-tip-classes.md)), so a liquid-relative move
(`heightFrom: 0` = `"Liquid"`) over a reservoir on those pods has no surface to reference
and fails. Bottom-referenced also composes correctly with the section geometry documented
here (the max-volume and top-section-depth columns above). For the enclosing item shape and other
height keys see [Transfer](../../step-documentation/11-Transfer.md); to have the
engine track reservoir starting/dead volumes so heights and volume checks stay accurate,
declare the section volumes in [Instrument Setup](../../step-documentation/03-Instrument-Setup.md).

```jsonc
// Source item aspirating from a reservoir, bottom-referenced:
{
  "labwareClass": "Reservoir",
  "position": "P3",
  "liquidType": "Well Contents",
  "volume": "150",
  "overrideHeight": true,
  "heightFrom": 1,          // 1 = "Bottom"; 0 = "Liquid" would require LLS (Span-8 only)
  "height": 1.0,            // mm above the section bottom
  "pattern":       { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true] },
  "selectionInfo": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true] }
}
```

## Titer Plates

TiterPlate labware classes model an addressable `WellsX × WellsY` grid. Numeric and class-name values below are reproduced exactly from each class's default project-item properties; Description text is reproduced with terminal punctuation normalized. Per-well physical dimensions (heights, spans, spacings) live in the labware class definition, viewable in the Labware Type Editor, and are omitted from this table.

| Class name | Wells | Max vol (µL) | Well shape | Default lid | Stacks on | Pods | Description |
|---|---|---|---|---|---|---|---|
| `AB384WellReactionPlate` | 24×16 | 40.0 | Round | `TiterplateLid` | self | any | 384 well thermal cycler style plate |
| `AB_0661` | 12×8 | 2200.0 | Rectangle | — | self | any | ABGene 0661 (individual) and 0778 (bulk) deepwell storage plates. |
| `AB_0765` | 12×8 | 830.0 | Round | — | self | any | Abgene 0765 or 0865 polypropylene microplate, half as tall as a standard deep well plate, with 96 conical-bottomed wells. |
| `AB_0932` | 12×8 | 2070.0 | Rectangle | — | self | any | AbGene 0932 deepwell storage plate. |
| `AB_1127` | 12×8 | 1200.0 | Rectangle | — | self, `Axygen_P96450VC` | any | AbGene 1127, 96 well plate, 1.2ml square well with round bottom. |
| `Axygen_P96450VC` | 12×8 | 500.0 | Round | — | self | any | Axygen P-96-450VC V-bottom microplate, clear, sterile. |
| `BCDeep96Round` | 12×8 | 1317.9726 | Round | `TiterplateLid` | — | any | Beckman Coulter microplate with 96 deep, round-bottomed wells. |
| `BCFlat96` | 12×8 | 362.76 | Round | `TiterplateLid` | self | any | Beckman Coulter microplate with 96 round, flat-bottomed wells. |
| `Bio_RadPCR384` | 24×16 | 38.0 | Round | — | self | any | Bio-Rad Hard-Shell 384-well PCR Plate (formerly MJ Research): thin-wall, skirted. |
| `Bio_RadPCR96` | 12×8 | 186.0 | Round | — | self | any | BioRad HardShell Low Profile Fully Skirted PCR Plate. |
| `CorningCostar12` | 4×3 | 6900.0 | Round | `TiterplateLid` | `CorningCostarLid` | any | Corning Costar Cell Culture Microplate, 12 wells. |
| `CorningCostar24` | 6×4 | 3480.0 | Round | `TiterplateLid` | `CorningCostarLid` | any | Corning Costar Cell Culture Microplate, 24 wells. |
| `CorningCostar48` | 8×6 | 1620.0 | Round | `TiterplateLid` | `CorningCostarLid` | any | Corning Costar Cell Culture Microplate, 48 wells. |
| `CorningCostar6` | 3×2 | 16800.0 | Round | `TiterplateLid` | `CorningCostarLid` | any | Corning Costar Cell Culture Microplate, 6 wells. |
| `CostarFlat384Square` | 24×16 | 112.0 | Rectangle | `CostarFlat384SquareLid` | self | any | Costar microplate with 384 square, flat-bottomed wells. |
| `Eppendorf_twintec96_FullSkirt` | 12×8 | 220.0 | Round | — | self | any | Eppendorf twin.tec PCR Plate 96, skirted. |
| `Greiner384ConePP` | 24×16 | 130.0 | Rectangle | `Greiner384Lid` | self | any | Greiner polypropylene microplate with 384 square, V-bottomed wells. |
| `Greiner384ConePPDeep` | 24×16 | 240.0 | Rectangle | `Greiner384Lid` | self | any | Greiner polypropylene microplate with 384 round, deep, V-bottomed wells. |
| `Greiner96ConePS` | 12×8 | 230.0 | Round | `Greiner384Lid` | self | any | Greiner polystyrene microplate with 96 round, V-bottomed wells. |
| `Greiner96RoundDeep` | 12×8 | 1200.0 | Round | — | — | any | Greiner polypropylene microplate with 96 deep, round, round-bottomed wells. |
| `Greiner96RoundDeepSquare` | 12×8 | 2300.0 | Rectangle | — | — | any | Greiner polypropylene microplate with 96 deep, square wells. The product is specified v-bottom, but the labware definer cannot express a v-bottom, so this class is defined with a round bottom (tested as acceptable). |
| `Greiner96RoundPS` | 12×8 | 300.0 | Round | `Greiner384Lid` | self | any | Greiner polystyrene microplate with 96 round, U-bottomed wells. |
| `GreinerCELLSTAR12` | 4×3 | 6550.0 | Round | `TiterplateLid` | `GreinerCELLSTARLid` | any | Greiner 12 Well Cell Culture Multiwell Plate. |
| `GreinerCELLSTAR24` | 6×4 | 3300.0 | Round | `TiterplateLid` | `GreinerCELLSTARLid` | any | Greiner 24 Well Cell Culture Multiwell Plate. |
| `GreinerCELLSTAR48` | 8×6 | 1700.0 | Round | `TiterplateLid` | `GreinerCELLSTARLid` | any | Greiner 48 Well Cell Culture Multiwell Plate. |
| `GreinerCELLSTAR6` | 3×2 | 16400.0 | Round | `TiterplateLid` | `GreinerCELLSTARLid` | any | Greiner 6 Well Cell Culture Multiwell Plate. |
| `ILS_2mlDeepSquare` | 12×8 | 2200.0 | Rectangle | — | self | any | Irish Life Science microplate with 96 deep, square, round-bottomed wells, 2.2 mL capacity. The external wall taper does not support an undergrip for moving stacks; when stacking, adjust the gripper offset to pick up above the skirt. Dead volume 10 µL per well. |
| `BCDeep96Square` | 12×8 | 2200.0 | Rectangle | — | — | Multichannel, Span-8 | Beckman Coulter microplate with 96 deep, square, V-bottomed wells. |
| `BCI_12_Strip` | 12×8 | 344.0 | Round | `titerplatelid` | — | Multichannel, Span-8 | Beckman Coulter microplate with 96 round, flat-bottomed wells. |
| `GreinerFlat1536Square` | 48×32 | 12.0 | Rectangle | `GreinerFlat1536SquareLid` | self | Multichannel | Greiner microplate with 1536 square, flat-bottomed wells. |
| `WashStation` | 12×8 | 1000000.0 | Round | — | — | Multichannel, Span-8 | — |
| `WashStation384` | 24×16 | 1000000.0 | Round | — | — | Multichannel | — |
| `WashStationSpan8` | 1×8 | 1000000.0 | Round | — | — | Span-8 | — |

All titer plates carry `Can Pipette` and `Moveable` except `BCDeep96Square`, `WashStation`, `WashStation384`, and `WashStationSpan8`, which carry `Can Pipette` but lack `Moveable`. A class without `Moveable` cannot be repositioned by a Move Labware step; targeting one fails at enqueue with `Labware of type <class> cannot be moved.` All but the three wash stations also carry `Standard TiterPlate Size`; the wash stations carry `Do Not Report` instead, so they do not appear in reports. The Stacks on column lists the classes an instance may be placed on top of; `self` means another instance of its own class.

## Tip Boxes

A tip box class is what a pipetting step's `tipLocation` names. Each default box holds exactly one tip class; the box name is the tip class name with `T` replaced by `BC`. Tip capacity, filtering, LLS capability and bore live in [Catalog: Tip Classes](catalog-tip-classes.md).

The suffixes carry over from the tip class: the number is the tip's nominal capacity in µL, `F` is filtered, `_LLS` is the LLS-capable variant, `_WB` is wide-bore, and `_384` is a 384-format box for a 384-channel head. See [Catalog: Tip Classes](catalog-tip-classes.md#naming-scheme) for the full decoding and the per-tip properties.

All 24 default boxes are Beckman Coulter, use lid class `TipBoxLid`, have round wells that each hold one tip, carry the `Moveable` and `Standard TiterPlate Size` characteristics (a tip box does not carry `Can Pipette`), and are not stackable on other labware. Nine also carry `Do Not Report`: `BC190F`, `BC190F_LLS`, `BC190F_WB`, `BC230`, `BC230_WB`, `BC40F`, `BC50F`, `BC80`, and `BC90`.

| Tip box class | Holds tip class | Tips | Pods |
|---|---|---|---|
| `BC1025F` | `T1025F` | 96 | any |
| `BC1025F_LLS` | `T1025F_LLS` | 96 | any |
| `BC1025F_WB` | `T1025F_WB` | 96 | any |
| `BC1070` | `T1070` | 96 | any |
| `BC1070_LLS` | `T1070_LLS` | 96 | any |
| `BC1070_WB` | `T1070_WB` | 96 | any |
| `BC190F` | `T190F` | 96 | any |
| `BC190F_LLS` | `T190F_LLS` | 96 | any |
| `BC190F_WB` | `T190F_WB` | 96 | any |
| `BC230` | `T230` | 96 | any |
| `BC230_LLS` | `T230_LLS` | 96 | any |
| `BC230_WB` | `T230_WB` | 96 | any |
| `BC40F` | `T40F` | 96 | any |
| `BC40F_LLS` | `T40F_LLS` | 96 | any |
| `BC50F` | `T50F` | 96 | any |
| `BC50F_LLS` | `T50F_LLS` | 96 | any |
| `BC80` | `T80` | 96 | any |
| `BC80_LLS` | `T80_LLS` | 96 | any |
| `BC90` | `T90` | 96 | any |
| `BC90_LLS` | `T90_LLS` | 96 | any |
| `BC25F_384` | `T25F_384` | 384 | Multichannel |
| `BC30_384` | `T30_384` | 384 | Multichannel |
| `BC40F_384` | `T40F_384` | 384 | Multichannel |
| `BC50_384` | `T50_384` | 384 | Multichannel |

## Tube Racks

Tube racks come in two deck footprints. The `Matrix96_*` and `SmallTubeRack_*` classes occupy a standard microplate footprint, about 12.8 × 8.5 cm. The eight `BCTubeRack_*` and `BCSeptaTubeRack_*` classes are oversize at 13.716 × 26.67 cm, roughly three microplate footprints deep, and sit on the dedicated `TubeRack` ALP, which provides a single position that does not allow gripping.

Height matters too. Most microplates are under about 4.5 cm tall. The oversize racks run 7.66 to 11.587 cm, and three of the four `SmallTubeRack_*` classes are standard-footprint but tall — 8.0 cm for the 10 mm and 12 mm racks, 10.5 cm for the 13 mm rack. Only the two `Matrix96_*` racks, at 3.556 and 4.55 cm, and `SmallTubeRack_Microfuge`, at 3.95 cm, stay in plate territory. Those same two `Matrix96_*` classes are also the only ones of the fourteen that carry `Moveable`; for the rest a Move Labware step cannot reposition them.

All tube racks carry `Can Pipette`. The four `BCSeptaTubeRack_*` classes are septum-capped and require piercing tips. For every `BCTubeRack_*` and `BCSeptaTubeRack_*` class the well geometry assumes the inner diameter noted in the table, and 2 mm is subtracted from the nominal tube height for height and volume calculations. The four `SmallTubeRack_*` classes are 24-position racks built on BCI 373661 with class-specific inserts and tubes.

| Class name | Tubes | Max vol/tube (µL) | Footprint | Height (cm) | Moveable | Pods | Notes |
|---|---|---|---|---|---|---|---|
| `Matrix96_1400uL` | 12×8 | 1380.0 | Standard plate | 4.55 | Yes | any | Thermo Scientific Matrix 1.4 mL 2D Barcoded Storage Tubes |
| `Matrix96_750uL` | 12×8 | 750.0 | Standard plate | 3.556 | Yes | any | Thermo Scientific Matrix 0.75 mL 2D barcoded storage tubes; height assumes septa in place, so remove septa when using non-piercing tips |
| `SmallTubeRack_Microfuge` | 6×4 | 1700.0 | Standard plate | 3.95 | No | any | Inserts BCI 373696, tubes BCI 356090 |
| `SmallTubeRack_10mm` | 6×4 | 4200.0 | Standard plate | 8.0 | No | any | Inserts BCI 373699, tubes VWR 60825-538 |
| `SmallTubeRack_12mm` | 6×4 | 6200.0 | Standard plate | 8.0 | No | any | Inserts BCI 373698, tubes VWR 60825-550 |
| `SmallTubeRack_13mm` | 6×4 | 10000.0 | Standard plate | 10.5 | No | any | Inserts BCI 373697, tubes VWR 60825-571 |
| `BCTubeRack_10mm` | 10×16 | 3552.0 | Oversize | 7.66 | No | Multichannel, Span-8 | 10 mm tubes, 75 mm; 8 mm assumed inner diameter |
| `BCTubeRack_12mm` | 8×16 | 5523.0 | Oversize | 7.66 | No | Multichannel, Span-8 | 12 mm tubes, 75 mm; 10 mm assumed inner diameter |
| `BCTubeRack_13mm` | 8×12 | 9043.0 | Oversize | 10.16 | No | Multichannel, Span-8 | 13 mm tubes, 100 mm; 11 mm assumed inner diameter |
| `BCTubeRack_15_5mm` | 6×8 | 13562.0 | Oversize | 10.16 | No | Multichannel, Span-8 | 15.5 mm tubes, 100 mm; 13.5 mm assumed inner diameter |
| `BCSeptaTubeRack_13x75mm` | 8×12 | 6668.0 | Oversize | 9.087 | No | Span-8 | 13 mm / 75 mm, 11 mm assumed inner diameter |
| `BCSeptaTubeRack_13x100mm` | 8×12 | 9043.0 | Oversize | 11.587 | No | Span-8 | 13 mm / 100 mm, 11 mm assumed inner diameter |
| `BCSeptaTubeRack_15_5x75mm` | 6×8 | 9983.0 | Oversize | 9.087 | No | Span-8 | 15.5 mm / 75 mm, 13.5 mm assumed inner diameter |
| `BCSeptaTubeRack_15_5x100mm` | 6×8 | 13562.0 | Oversize | 11.587 | No | Span-8 | 15.5 mm / 100 mm, 13.5 mm assumed inner diameter |

## Lids

All eight lid classes are 1×1 with `WellStyle: None` and zero well volume, and carry `Moveable` and `Standard TiterPlate Size`. All carry `Do Not Report` except `NuncLid`. `CostarFlat384SquareLid`, `Greiner384Lid`, and `GreinerFlat1536SquareLid` each name their own class as a stacking target, shown below as `self`. A blank Stacks on cell does not mean the lid is unusable: `TiterplateLid` and `TipBoxLid` are assigned through the plate's or box's default-lid setting rather than through a stacking list.

| Class name | Stacks on | Pods | Description |
|---|---|---|---|
| `CorningCostarLid` | `CorningCostar12`, `CorningCostar24`, `CorningCostar48`, `CorningCostar6` | any | Lid for Corning Costar cell culture plates (6-, 12-, 24-, 48-, 96-well) |
| `CostarFlat384SquareLid` | self | any | Lid from a Costar 384 microplate |
| `Greiner384Lid` | self | any | Lid from a Greiner 384 microplate |
| `GreinerCELLSTARLid` | `GreinerCELLSTAR12`, `GreinerCELLSTAR24`, `GreinerCELLSTAR48`, `GreinerCELLSTAR6` | any | Lid for Greiner CELLSTAR cell culture plates (6-, 12-, 24-, 48-, 96-well) |
| `NuncLid` | — | any | Clear universal plate lid, no corners cut. |
| `TipBoxLid` | — | any | Lid from a Beckman Coulter tip box |
| `TiterplateLid` | — | any | Lid from a Beckman Coulter microplate |
| `GreinerFlat1536SquareLid` | self | Multichannel | Lid from a Greiner 1536 microplate |

## Reservations

Two reservation classes hold a deck position without providing pipettable wells. `SwapSpace` reserves a position for temporary storage during labware movement; `TipLoad` reserves a position for tip loading when tip boxes are provided from an external device. Both are 12×8 on a standard titer-plate footprint, work on any pod, and carry `Do Not Report`. Neither carries `Can Pipette` or `Moveable`.

Reservations are normally placed and managed by the software, not by a method author: you typically should **not** author a reservation class in an Instrument Setup `deckItems` list. Treat them as classes you may *see* in an export rather than ones you place by hand.
