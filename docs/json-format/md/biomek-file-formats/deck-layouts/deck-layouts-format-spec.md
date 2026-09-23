# Deck Layouts Format Specification

**What this document is.** How to read a deck out of a Biomek Instrument Settings JSON export.
The deck lives inside the export as `deckLayouts.<deckKey>` — this spec defines the envelope
around it, the per-deck object, and the per-position record shape (fields, types, and the
`$id`/`$ref` reference model). Values shown against the reference export are copied verbatim.

> **The export may carry more keys than are documented here, and it shares objects via
> `$id`/`$ref`.** The keys listed below are the ones you need to read to understand a deck —
> what ALPs are on it, where each position sits, what labware it accepts, and how it is wired
> to hardware. Any additional key you see in a real export is internal book-keeping; treat it
> as opaque and ignore it.

---

## What a deck is

A **deck** is a **permanent, reusable arrangement of ALPs** (Automated Labware
Positioners) bolted to a Biomek instrument at **fixed physical coordinates**. It
is built once in the **Deck Editor** utility (`Utilities ▸ Instrument ▸ Deck
Editor`) and shared across many methods. Each ALP occupies one or more numbered
**positions** (`P1`, `P2`, … plus named fixtures like `WS1`, `TR1`). The common
ALP types included with Biomek are listed in [ALP Catalog](./catalog-alps.md).
Additional ALP types beyond this list may appear on a given system (for example
add-on hardware or integrated devices):

**Permanent deck (this format) vs. temporary labware (NOT this format):**

| | Permanent deck | Temporary labware |
|---|---|---|
| Built in | **Deck Editor** utility | **Instrument Setup** method step |
| What it defines | ALPs at fixed coordinates (the board) | Plates/tip boxes/reservoirs placed on ALPs for one run (the pieces) |
| Lifetime | Permanent, reused by every method | Per-run, per-method |
| Serialized as | `deckLayouts` in an **Instrument Settings** export (this doc) | `deckItems` in **method** JSON (a different format) |

One-liner: **the Deck Editor builds the board (ALPs); Instrument Setup places
the pieces (labware) on it.** This document describes the *board*. When a method
authors labware placement it writes `deckItems` keyed by the position names that
this deck defines — it never edits the deck. `deckItems` is a method-JSON
structure and is specified elsewhere: see
[Instrument Setup](../../step-documentation/03-Instrument-Setup.md)
for the step that writes it and
[Concept Guide: Labware & Position Resolution](../../concept-guides/02-labware-and-position-resolution.md)
for how a position name resolves to labware at run time.

**Why this format is useful:**

- To turn a raw exported deck into a clean `position → ALP type → coordinate →
  accepted-labware` table.
- To know which positions exist, i.e. which `Position` / `where` values are
  valid when authoring a method against this instrument.
- To understand that placing labware is temporary and happens via Instrument
  Setup (`deckItems`), not by changing the deck.

The illustrative resolved views under [`samples/`](./samples/) show one of
these reads applied to each default deck included with Biomek — same key names as the raw
export, with `$ref` pointers dereferenced and the internal book-keeping keys
elided.

---

## Where deck data comes from: the Instrument Settings JSON export

Decks are **not** exported as standalone files. They live inside a **"Biomek
Instrument Settings"** JSON export.

The examples throughout this document are drawn from the **i7 Hybrid** default configuration
(five decks: `dualMultichannel`, `hybrid`, `multichannel`, `span8`, `standard`).
The other six representative configurations cover i3, i5 Multichannel, i5 Span-8,
i7 Multichannel, i7 Span-8 (disposable-tip), and i7 Dual-Multichannel — see the [pod-settings spec](../instrument-settings/pod-settings-format-spec.md)'s
"Representative instrument configurations" section for the roster.

The `standard` deck is included as a read-only default. It is never the actual
deck being used and should not be selected in an **Instrument Setup** step.

### Top-level envelope

```jsonc
{
  "format": "Biomek Instrument Settings",
  "formatVersion": "1.0",
  "podSettings": { ... },
  "deckLayouts": { ... }
}
```

The category names are fixed: the
category `"Deck Layouts"` maps to the JSON key **`deckLayouts`**. Both
category keys — `podSettings` and `deckLayouts` — are
**optional**: a category appears only if the user selected one or more objects
of that type at export time, so be prepared for an Instrument Settings file that
omits `deckLayouts` entirely.
The reference file has `podSettings` + `deckLayouts`.

### `deckLayouts`

`deckLayouts` is an object keyed by **deck name** (camelCased — see below). The
reference export `biomek-i7-hybrid.json` contains five: `dualMultichannel`,
`hybrid`, `multichannel`, `span8`, `standard`. Each deck object has:

| Key | Type | Meaning |
|---|---|---|
| `name` | string | Original deck name, un-camelCased (e.g. `"Standard"`, `"DualMultichannel"`) |
| `isDefault` | bool | Whether this is the instrument's default deck |
| `isReadOnly` | bool | Read-only (factory) deck vs. user-editable |
| `groups` | object | ALP hardware definitions, keyed by group name — **all `$ref`** (see reference model) |
| `positions` | object | The placed positions, keyed by position name — the authoritative per-position records |

### Finding and using the instrument's default deck

An instrument defines one or more decks, and one of them is its **default**. In the export, the
default deck is the deck object under `deckLayouts` whose `isDefault` is `true`. Reading the export
first is the reliable way to know both **which deck names an instrument accepts** (the keys of
`deckLayouts`) and **which one is the default**.

- **Find the default:** scan `deckLayouts` for the entry with `"isDefault": true`; that entry's name
  is the instrument's default deck.
- **Author against it:** put that deck name in the Instrument Setup step's `layout` key and use it as
  the method's default deck (name matching is case-insensitive). Targeting a different deck the same
  instrument defines is fine — just use that deck's name instead. A `layout` value the instrument
  does not define fails at enqueue. Do not use multiple decks in the same method.

You need the instrument's **Instrument Settings JSON export** to determine its decks and its default.
If you do not have that export on hand, produce one first — in the Biomek software,
*File ▸ Export ▸ Instrument Settings ▸ Save As JSON* — and read the default deck from it as above;
without it there is no way to know which deck names a given instrument will accept.

---

## Key-naming and type conventions (must understand to parse)

These three rules are applied uniformly by the shared provider stack
(`PersistenceFormats/TypeProviders/`) and explain everything that looks odd in
the raw JSON.

### 1. CamelCase key conversion

Every dictionary key is passed through `JsonNamingPolicy.CamelCase`, which lowercases only the
first PascalCase word and preserves the rest verbatim (see
[Base Keys & Serialization Conventions](../../step-documentation/00-Base-Keys-and-Conventions.md#1-key-name-casing-spelling-global-rule) §1). So:

- Field names: `ALPType → alpType`, `PermanentLabware → permanentLabware`,
  `Positions → positions`, `X1 → x1`, `Framed1 → framed1`,
  `TemplatePositionName → templatePositionName`.
- **Position/deck keys too:** `P1 → p1`, `S1 → s1`, `WS1 → wS1`, `TL1 → tL1`,
  `TR1 → tR1`, `W1 → w1`; deck `Standard → standard`,
  `DualMultichannel → dualMultichannel`. Verified in the reference file
  (`positions` keys of the `standard` deck: `p1`…`p30`, `tL1`…`tL5`, `tR1`,
  `tR2`, `w1`, `wS1` — 39 keys).

Method `deckItems` use the same convention (e.g. `deckItems.p5`), so the
position keys line up between deck and method JSON — see
[Instrument Setup](../../step-documentation/03-Instrument-Setup.md).

### 2. `_biomekType` type tag

Objects that wrap a Biomek aggregate class carry a `_biomekType` string. A plain dictionary has **no**
`_biomekType` (kept concise). Friendly names seen in a deck export:

| `_biomekType` | Where it appears in a deck |
|---|---|
| `deckPosition` | every entry under `positions` |
| `labware` | permanent labware on a wash station (`position.labware`) |
| `device` | positions wired to a device — carries an extra `_deviceType` naming the device class (see "The `device` block") |
| *(none)* | the deck object, `groups`, `characteristics`, `required`, `pins`, `regions`, … (plain dictionary — no `_biomekType`) |

### 3. `$id` / `$ref` reference model — resolve every reference

The instrument object graph is cyclic and shares objects, so the serializer
uses reference tracking. A shared object is **materialized once**
(first occurrence, tagged with `$id` = a JSON Pointer into the document) and
appears everywhere else as `{"$ref": "<same pointer>"}`.

In a deck this appears in three places. An interpreter that does not follow
`$ref` produces an incomplete deck.

1. **Top-level `groups.*` are all `$ref`.** The group object is materialized
   inline under its host position. Example from the `hybrid` deck:
   `groups.static1x1_1` = `{"$ref":
   "#/deckLayouts/hybrid/positions/p1/group"}`.
2. **`position.group`** holds the inline group object (with `$id`) on the host
   position; `position.group.positions.<name>` lists the member positions,
   back-referencing the host via `$ref`.
3. **Multi-position ALPs (e.g. `Static1x3`) hide their secondary members.** A
   `Static1x3` group spans three positions (e.g. `p3`, `p4`, `p5`). Only one
   member — the *host* — carries a full record at `positions.<key>`; the full
   records for the other two are materialized *inside*
   `positions.<host>.group.positions`, and their top-level entries are pure
   `{"$ref": "#/deckLayouts/hybrid/positions/p3/group/positions/p4"}`.
   In the reference `hybrid` deck, the secondary members of each `Static1x3`
   ALP appear as pure `$ref` at the top level.

   The host is whichever member key sorts first in the export's key order, which
   is ordinal string order, **not** numeric order. In the reference `hybrid`
   deck the `p8`/`p9`/`p10` ALP is hosted by `p10` (because `"p10" < "p8"` as
   strings), so the full `p8` and `p9` records live under
   `positions.p10.group.positions`. Never assume the lowest-numbered member is
   the host — follow the `$ref`.

Resolution rule: if `positions.<key>` has only a `$ref`, dereference the pointer
(a `#/...`-rooted JSON Pointer) to get the real record before reading its
fields.

Pointer segments carry two-stage escaping — RFC 6901 (`~` → `~0`, `/` → `~1`)
wrapped in full URL escaping — but no `$id`/`$ref` value in any of the seven
default instrument exports contains a `~` or a `%`, so plain `/`-splitting
resolves every pointer in them. Implement the full unescape only if you want to
be strict about arbitrary keys. The [pod-settings spec](../instrument-settings/pod-settings-format-spec.md) states the
same rule for the shared envelope.

---

## The position record (`positions.<key>`)

The authoritative per-position object. Fields present in the reference file:

| Field | Type | Meaning / how to use |
|---|---|---|
| `_biomekType` | `"deckPosition"` | type tag |
| `name` | string | Display name, original casing (`"P1"`, `"WS1"`). Use this name to reference a position in method steps. |
| `baseName` | string | Family prefix (`"P"`, `"WS"`, `"TR"`) |
| `alpType` | string | **ALP type** — typically one of the common ALP-type names listed under "What a deck is", verbatim (`"Static1x1"`, `"Static1x3"`, `"WashStation96"`, `"TrashLeftSlide"`, …); other values may appear for add-on hardware or integrated devices |
| `templatePositionName` | string | Internal template cell (often `"A1"`); **not** a reliable grid coordinate. Ignore this value. |
| `x1`,`y1`,`z1` | number (cm) | **Pod 1** calibrated access coordinate, in **centimeters** |
| `x2`,`y2`,`z2` | number (cm) | **Pod 2** calibrated access coordinate, in **centimeters** |
| `xSpan`,`ySpan`,`zSpan` | number (cm) | Footprint extent |
| `framed1`,`framed2` | string | Framing state per pod (`"Not"` = not framed/calibrated). Methods should prefer to reference framed positions for a given pod when possible. |
| `permanentLabware` | bool | `true` ⇒ labware is permanently installed here (wash stations) |
| `labware` | object \| null | The installed permanent labware when `permanentLabware=true` (a `_biomekType:"labware"` object with `class`, `properties.liquidtype`, `volumeType`); `null` otherwise |
| `required` | object | **What labware this position accepts** — keys are constraint names (e.g. `"standard TiterPlate Size"`, `"no plates allowed"`); values `null` |
| `characteristics` | object | Capability flags — keys are characteristic names (e.g. `"multichannel Can Load Tips"`, `"span Can Load Tips"`, `"fixed-8 Can Load Tips"`, `"wash Station"`, `"can Discard Tips"`, `"disallow ManualTeach"`). The values are **always `null`** in an instrument-settings export; the capability is signaled by the **presence of the key**, not by its value — read the key names, and ignore the `null`. A characteristic states a **capability, not reachability**: a pod that a position is built to serve may still be unable to reach it on a given instrument — see [Reachability and Access](./reachability-and-access.md). |
| `adaptorPlate` | object | Framing/setup instruction (`bitmap`, `caption`/`message`, `visible`, `x`/`y`/`z`, and on tip-load positions `frameWithGripper`/`frameWithTips`) — e.g. WS1: *"Lift the wash station off its base and replace it with the AccuFrame."* Irrelevant to method authoring. |
| `group` | object | The ALP group this position belongs to (inline with `$id`, or `$ref`) — see "The group record" below |
| `device` | object \| null | Device wiring; `null` on ordinary positions — see "The `device` block" below |
| `deviceIndex` | int | Index of this position within the device's `deckPositions`; `-1` when there is no device |
| `canCopyDevice` | bool | Present (and `true`) on trash positions. Marks the `device` block as fixed to the deck — carried with the deck itself rather than bound to a separately-installed device |
| `labwareOffsets` | object | Per-labware-class placement corrections. Keys are camelCased labware class names (`"bC1025F"`, `"agilentReservoir"`, …); each value is `{"x":…,"y":…,"z":…}` in cm. `{}` on most positions; populated on tip-load (`tL*`) and reservoir positions |
| `obstacles` | object | Present on trash positions. The ALP's walls, keyed `back`, `front`, `left`, `right`; each value has `position` (a `$ref` back to the owning position) plus `xOffset`/`yOffset`/`zOffset` and `xSpan`/`ySpan` in cm, relative to the position origin |
| `wasteX` | number (cm) | Present on wash stations. X distance from the center of the wash points to the center of the drain hole |
| `tipDisposalOffsetX/Y/Z`, `span8TipDisposalOffsetX/Y` | number (cm) | Present on trash positions. Where each pod type releases tips relative to the position |
| `minSafeHeight`, `offsetX/Y/Z`, `labwareX/Y/Z`, `podAccessResource`, `deck`, `stack`, `rT_Labware`, `_RT_Stack` | various | Runtime/geometry internals; not needed for a position→ALP map |

Keys are omitted when they carry no value, so a position record is not a fixed
shape: read defensively and treat a missing key as "not applicable to this ALP".

**Tip-load characteristics are asymmetric.** The two tip-load capability keys do not appear on
the same set of positions: `multichannel Can Load Tips` is present **only** on the dedicated
tip-load (`tL*`) positions, while `span Can Load Tips` is present on those **and** on every
plain `p*` position. So a Multichannel tip box is confined to the `tL*` positions, whereas a
Span-8 tip box may be placed on any `p*` position. Because these are capability flags, the
`span Can Load Tips` on a `tL*` position does not mean the Span-8 can *reach* it — on a two-pod
instrument it often cannot. See
[Reachability and Access](./reachability-and-access.md#tip-loading-capability-is-not-reachability) §"Tip loading: capability is not
reachability" and the i7 table in [Pod Reach Envelopes (i7 worked
example)](./pod-reach-envelopes.md).

Note that for the Multichannel pod, if a tip box is placed on a position that is
**not** a TipLoad1x1 ALP, the software will automatically move it to the
TipLoad1x1 in order to load tips, if possible.

### The `device` block

`device` is an **object**, not a string, and binds a position to installed hardware. On an ordinary
position it is `null`. Positions wired to hardware (trash bins and slides, heaters/shakers, and other
device ALPs) carry a full `_biomekType: "device"` object; `_deviceType` names the device class. The
`standard` deck's `tR1` reads:

```json
{
  "_biomekType": "device",
  "_deviceType": "Biomek5.TrashDevice",
  "deckPositions": [
    { "$ref": "#/deckLayouts/standard/positions/tR1" }
  ],
  "isReadOnly": true,
  "name": "Trash"
}
```

`name` is what a step refers to. `deckPositions` is an array of `$ref` back to every position the
device covers, so the link is bidirectional; `deviceIndex` on the position is its slot in that array.
`isReadOnly` marks a factory-defined device. The illustrative sample views collapse `device` to its
`name` string.

Devices are installed and configured on the instrument, not created by a method. To find what yours
has, scan the export for positions whose `device` is not `null` — an ALP that *can* host a device
(orbital-shaker or heat/cool platform) is not the same as one being installed on it.

The default-instrument samples here carry only a trash device; a system with a shaker, Peltier, or
bar-code reader shows other `_deviceType` values at those positions. The i3 sample has no `device`
block on any position — its tip trash is a plain ALP.

#### Which steps address a device by position, and which by name

What you write depends on the step:

| Step | You write | Resolved from |
|---|---|---|
| [INHECO Peltier](../../step-documentation/10-INHECO-Peltier.md) | a **deck position** (`position`) | the device object on that position |
| [Device Action](../../step-documentation/09-Device-Action.md) | a **device name** (`device`) | the pipettor's device collection |
| [Fly-By Read](../../step-documentation/46-Fly-By-Read.md) | a **device name** *and* a deck position | the collection, then checked for a position association |
| [SILAS](../../step-documentation/71-SILAS.md) | a **module name** (`module`) | the configured SILAS modules |

Peltier does not take a device name: you name the position, and the step reads the device off it at
enqueue. It fails in one of three ordered ways, and only the first is fixable by editing the
method:

```
P3 is not a valid position name for the current deck.
P3 does not have a device associated with it.
Location does not have an Inheco Peltier device associated with it.
```

The first means the name is wrong, or right for a different deck than the one resolved. The second
means the position is real but nothing is wired to it — **not fixable from the method**: placing an
ALP there does not associate a device with it, and no method key binds one. The third means a device
is associated but is not a Peltier; the message says `Location` literally rather than naming your
position.

Fly-By Read is the opposite: a device that exists but has no position association gives

```
The device named "<name>" is not associated with a deck position. Please use the deck editor to
associate "<name>" with a position
```

Both are instrument-configuration messages — check the export before authoring a device step.

SILAS is outside this model. A module is not read off a position's `device` block; the modules are
a separate configured collection.

### The group record (`positions.<key>.group`)

The group is the ALP hardware definition. It is materialized inline (with `$id`)
on its host position and appears as `$ref` everywhere else, including under
`deckLayouts.<deck>.groups`.

> **You'll see a `group` on every position in your real export.** It is the shared ALP
> definition (a *position group*) linked across the ALP's positions via `$id`/`$ref` — per-ALP
> hardware data repeated by reference, not per-position placement. The illustrative views under
> [`samples/`](./samples/) omit it; for what each ALP type is, see [ALP Catalog](./catalog-alps.md).

| Field | Type | Meaning / how to use |
|---|---|---|
| `name` | string | Group instance name, original casing (`"Static1x1_1"`, `"TrashLeftSlide_1"`) |
| `type` | string | The ALP type — same value as the member positions' `alpType` |
| `column`, `row` | int | **Grid indices of the ALP on the deck.** Read these; do not compute letters |
| `positions` | object | Member positions, keyed by position name — the host as `$ref`, the secondaries as full inline records |
| `categories` | object | Set membership; keys are `basic`, `wash`, `trash`, `span8`, `tubes`, `devices` (values `null`). A group may carry more than one, e.g. `{"span8": null, "trash": null}` on a trash slide |
| `simulatorRenderStyle` | string | How the simulator draws the ALP. Irrelevant to method authoring. |
| `pins` | object | The ALP's **deck mounting-point footprint**, not a well grid. `colCount`/`rowCount` = mounting points spanned by the pin bounding box; `offsetX`/`offsetY` = cm from the ALP origin to the top-left corner of that bounding box; `identifier` = X distance in *pitches* from the left edge of the bounding box to the pointing feature on the ALP's front face |
| `regions` | array | Where on the deck the ALP may be placed, expressed against the deck's own grid. Each entry has `offsetFromDeckBackRow`, `offsetFromDeckFrontRow`, `offsetFromDeckLeftCol`, `offsetFromDeckRightCol` — 0-based counts of mounting points inward from each deck edge. Each axis needs two of three keys; when only one edge offset is given, `absColCount` / `absRowCount` supplies the span (end-columns/rows inclusive), as on the trash slides |
| `trashLeftRightWallZSpan` | number (cm) | Trash groups only. Height from the top of the deck plate to the top of the trash bin |
| `x1`,`x2`,`y1`,`y2`,`z1`,`z2`,`xSpan`,`ySpan`,`zSpan` | number (cm) | The ALP exterior. These differ from the member position's own coordinates, which describe the labware seat inside the ALP |

### Coordinates and grid

- **Units are centimeters**, stored verbatim (no conversion). Coordinates are
  calibration data: read them from the file you were given, never from another
  instrument's export.
- **Grid indices** come from the group's explicit integer `column` / `row` —
  read them; do **not** try to recompute letters.
  The Deck Editor UI shows only *sparse* column letters
  (`A, F, M, T, AA, AH, AO, AV, BC, BJ, BQ`) and rows (`5,10,15,20,25,30`) as a
  display overlay; those labels are coarser than the raw indices
  and are not needed to identify a position.

---

## The `samples/*.deck.md` files

The seven files under [`samples/`](./samples/) — one per default deck included with Biomek —
are **illustrative resolved views**, not raw dumps. Each file mirrors the shape
of `deckLayouts.<deckKey>` in the export, with `$id`/`$ref` dereferenced inline
and the hardware-internal and geometry-only keys elided so the deck's
`positions` map fits on the screen. Coordinates are copied verbatim from the
export (**cm**, not mm); `alpType`, labware `class`, device `name`, `required`
and `characteristics` are copied verbatim.

Read them as **guides to the keys**, not deterministic parser output — a real
export carries `$id`/`$ref` object sharing and additional book-keeping keys
that these views deliberately do not show. When a value is load-bearing (a
coordinate, a labware class, a device name), always confirm against the JSON
export itself.
