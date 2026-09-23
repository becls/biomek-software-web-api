# Project Items Format Specification

> **Planned / future format — not a current Biomek export.** Biomek has **no JSON export for
> project items** today; its JSON formats cover only methods (import and export) and instrument
> settings (export only). The catalogs described here are a **forward-looking preview** built from
> the default project configurations, not something the released software emits — treat them as
> reference, not as a guide to authoring importable project items today.
>
> **These are the defaults, and project items are broader than they look.** The companion
> `catalog-*.md` files are *derived summaries* of the default Biomek project
> configurations; the values they show are the **defaults** included with Biomek. Project items
> include labware classes, labware patterns, liquid types, tip classes, and **pipetting techniques
> and templates**. Many customers maintain their own project items in addition to the
> defaults.

## Purpose

These catalogs give a readable, per-category summary of the project-item definitions Biomek includes by
default — the labware classes, tip classes, liquid types, well patterns, pipetting techniques, and
templates a method can reference. A project-item definition is reusable metadata: methods
instantiate and reference it at runtime (many labware instances can share one labware class).

Biomek itself has no JSON export for project items. The catalogs are therefore a derived summary, not an importable export, and — apart from the technique catalog, which mirrors the real technique-serializer shape — they do not follow the envelope conventions of Biomek's own JSON (`format`/`formatVersion`, `$id`/`$ref`, and so on).

## Companion Catalogs

The defaults are broken out by category into the six per-category catalog files below.
Each catalog aggregates one category across the default instrument projects, so labware classes, tip
classes, techniques, and so on each live in one place:

| Catalog | Shape | What it lists |
|---|---|---|
| `catalog-labware-classes.md` | prose + a table per item type | the default labware classes |
| `catalog-labware-patterns.md` | prose + reference form | the default 384-well quadrant patterns |
| `catalog-liquid-types.md` | prose | the default liquid types (Water, Serum) |
| `catalog-tip-classes.md` | prose + one table | the default tip classes |
| `catalog-techniques.md` | one JSON block per technique | the default pipetting techniques |
| `catalog-templates.md` | prose | the default pipetting templates |

## Item shape and conventions

The technique catalog, which lists items as JSON, uses a **flat per-item block** — there is no
project/category envelope. The labware and tip-class catalogs describe the same per-item shape but
present it as tables. Each item carries the fields relevant to its category:

- **`itemName`** — the item's key (for example `T1025F`, `SeptaFluted`).
- **`hardwareCompatibility`** — present on every cataloged item (see below).
- **`properties`** — the method-authoring-relevant values for the item (per-category sets below).
- Labware classes additionally carry **`itemType`** (`TiterPlate`, `TipBox`, `TubeRack`,
  `Reservoir`, `Lid`, or `Reservation`), **`characteristics`** (presence flags such as `Can Pipette`
  and `Moveable`), and **`stackableOnLabwareClasses`** (the classes this class may stack on,
  including itself when it self-stacks). Tip classes and techniques carry none of these three. In
  `catalog-labware-classes.md` these three surface as the per-type sections, the per-section prose
  notes, and the **Stacks on** table column respectively.

Techniques are the exception to the flat-item convention: the technique catalog emits the real
serialized technique object — top-level `name` (not `itemName`), the technique's own nested blocks
(aspirate / dispense / mix / context / general) rather than a `properties` wrapper, and
`hardwareCompatibility` shown as a catalog annotation above each block rather than a field inside
it.

Liquid types and templates are documented as **prose** — liquid behavior is better viewed in the
Liquid Type editor, and templates are step-tree definitions best viewed in the Template Editor.
Labware patterns are documented as prose plus the **by-name reference form** used in a step
(`localPattern: false` + `referencedPattern`).

### hardwareCompatibility

`hardwareCompatibility` is **pod-based**: a list drawn from `Multichannel`, `Span-8`, and `Fixed-8`.
Chassis size does not affect it, and a hybrid instrument supports the union of its pods. It is a
catalog annotation, **derived** from which default projects include the item — not an
intrinsic, importable field. Each pod tag is derived from the single-pod default project that uses
that pod:

| Default single-pod project | Pod |
|---|---|
| i3 | Fixed-8 |
| i5 Multichannel | Multichannel |
| i5 Span-8 | Span-8 |

An item's `hardwareCompatibility` is the set of those pods whose project includes the item, or
`["any"]` when all three do. The i7 Hybrid project (which has both a Multichannel and a Span-8 pod)
is **not** used to add pod tags, because membership there cannot distinguish which pod an item is
for.

### Properties by category

Each item emits only the keys its own definition carries.

| Category | `properties` keys |
|---|---|
| Labware Classes | `Description`, `PartNumber`, `DefaultLidClass`, `Height`, `XSpan`, `YSpan`, `WellsX`, `WellsY`, `WellStyle`, `WellVolume`, `WellDepth`, `WellXSpacing`, `WellYSpacing`, `Wells(0).MaxVolume` (the maximum volume of the representative well, index 0; the key name mirrors the source property path), plus `Refills` (Reservoir classes) and `RequiresPiercing` (septum-capped tube-rack classes). The class type is carried in `itemType`, not repeated in `properties`. These are the keys that appear across labware classes; a given class emits only the subset its type carries — e.g. Reservation classes emit only `Description`/`Height`/`XSpan`/`YSpan`/`WellsX`/`WellsY`, Lid classes omit `DefaultLidClass` and well-depth/max-volume keys, and Reservoir classes omit most well-geometry keys. `catalog-labware-classes.md` carries the method-authoring-relevant subset of these as table columns — `Wells(0).MaxVolume` as the max-volume column, `WellsX`/`WellsY` as the wells column, `WellStyle` as the well-shape column, `DefaultLidClass` as the default-lid column — and omits the pure-geometry keys. |
| Tip Classes | `AirCapacity`, `Capacity`, `Filtered`, `Fixed`, `LLS`, `Piercing`, `TipHeight`, `UseSystemTubing`. Every tip class emits all eight. `catalog-tip-classes.md` carries the first six as table columns and states the three uniform ones — `Fixed`, `Piercing`, `UseSystemTubing` — in prose, since only `Fixed100` and `SeptaFluted` differ. |
| Techniques | the real serialized technique object (`_biomekType: "technique"` plus the technique's nested aspirate / dispense / mix / context blocks) — see the technique catalog for the shape and a worked example |

Reading notes:

- `Refills` applies to `Reservoir`-type classes and no other, and is the **Refills** column in the
  reservoir quick reference; `RequiresPiercing` applies only to the septum-capped
  `BCSeptaTubeRack_*` classes, noted in prose in the Tube Racks section.
- In the underlying source data a liquid type's `isDefault` marks the project default (Water) and
  is absent, not `false`, on the others; the liquid catalog is prose-only and does not surface this
  field directly.
- The technique catalog shows a few techniques in full and the rest with their identifying fields
  only, to keep the page compact; open a technique in the Technique Editor for fully resolved values.

## Notes

- Project items are definitions, not runtime instances; methods consume them and instantiate
  labware/tip/liquid usage dynamically.
- Instrument-specific projects vary in which classes, patterns, and techniques they provide.
