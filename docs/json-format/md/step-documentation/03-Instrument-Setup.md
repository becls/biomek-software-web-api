# Instrument Setup

| Property | Value |
|----------|-------|
| stepType | `"Instrument Setup"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | Not restricted (default applies to all: i3, i5, i7) |

## Behavior Summary

The Instrument Setup step configures the method's runtime representation of the
instrument deck and the pods' tip status.

Multiple Instrument Setup steps may appear in a single method to change the deck configuration mid-run.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `layout` | string | — | No (valid to omit) | instrument's default deck | Which of the instrument's decks this step configures. **Recommended to set explicitly** rather than omit: omitting falls back to the instrument's default deck, so a method later run on an instrument whose default deck differs from the one it was written for can behave unexpectedly — and silently, because omission never fails at enqueue. Setting `layout` makes a wrong deck either fail loudly or run correctly. When present, the value must name a deck the target instrument actually defines; deck names are per-instrument configuration, **not** a fixed vocabulary, so read the name from that instrument's settings export rather than copying a literal out of an example. An unrecognized name fails with "Deck \<name\> is not defined," and a resolved empty value fails with "No deck selected." Matching is case-insensitive. See §`layout` below. |
| `deckItems` | object (dictionary) | — | No | Empty dictionary | Dictionary keyed by position name (e.g., `"p1"`, `"tL1"`, `"wS1"`). Each value defines what labware to place at that position. See §Enumerated / Constrained Values for value types. If missing, uses an empty dictionary — engine tolerates omission (no labware placed). |
| `pause?` | boolean | — | No | `true` | When true, inserts a runtime pause with a confirmation dialog showing the deck layout. Omission is tolerated and defaults to `true`. |
| `verifyPodSetup?` | boolean | — | No | `true` | When true, applies pod tip configuration from `podSetup` at enqueue time and includes pod status in the confirmation dialog. Omission is tolerated and defaults to `true`. |
| `barcodeInput?` | boolean | — | No | `false` | When true, displays a barcode input dialog at runtime for labware identification. In simulation mode, the barcode form is constructed but not displayed; the visible dialog only appears during a real instrument run. Engine tolerates omission. |
| `podSetup` | object (dictionary) | — | No | Empty dictionary | Contains pod tip configuration. In enqueue the pod-setup confirmation runs only when `verifyPodSetup?=true` **and** `podSetup` is present in the step's parameters; the presence check makes omission always safe (the confirmation is skipped). Not required even when `verifyPodSetup?` is true. |
| `splitterPosition` | integer | pixels | No | Half the UI panel height | UI-only. Persists the vertical splitter position in the step editor. Editor-written; not read at runtime. |
| `verifyDeckOnly` | boolean | — | No | `false` | When true, the step only verifies/pauses without modifying the deck or pod state. Skips labware placement and pod setup entirely. |
| `deckSetupLabwareTypeFilter` | string | — | No | `"<Any>"` | UI filter for the labware type combo box in the step editor. Editor-written; not read at runtime. |
| `definesInitialConfiguration?` | boolean | — | No | Not present | Marks this Instrument Setup as the method's initial-configuration step (used by the editor to determine which Setup step defines the starting deck). Editor-written; not read at runtime. Author as a JSON boolean (`true`), e.g. `"definesInitialConfiguration?": true`. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, `isPreconfigured`, `bitmap`, …) and the node-level `disabled` property apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

### PodSetup Sub-Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `leftHasTips` | boolean or null | — | No | `false` | Tri-state: `true` = set the left pod's model state to have tips of the specified type; `false` = set the left pod's model state to no tips loaded; `null` = leave left pod unchanged (but still report current state in confirmation). |
| `leftTipType` | string | — | No | `""` | Tip box class name for the left pod, given as the **bare** tip-box class name with no prefix (e.g., `"BC230"`) — the same form a Select Tips step uses, NOT a `LabwareClasses\\` / `TipClasses\\` path. Only meaningful when `leftHasTips` is `true`. |
| `rightHasTips` | boolean or null | — | No | `false` | Tri-state: `true` = set the right pod's model state to have tips of the specified type; `false` = set the right pod's model state to no tips loaded; `null` = leave right pod unchanged. |
| `rightTipType` | string | — | No | `""` | Tip box class name for the right pod, given as the **bare** tip-box class name with no prefix (e.g., `"BC230"`), NOT a `LabwareClasses\\` / `TipClasses\\` path. Only meaningful when `rightHasTips` is `true`. |

> **Single-pod vs dual-pod instruments:** On single-pod instruments (i3, i5 Span-8, i5 Multichannel, or a one-pod i7), `leftHasTips`/`leftTipType` configure Pod1 (the only pod); `rightHasTips`/`rightTipType` are silently ignored when Pod2 is absent. On a dual-pod instrument (an i7 carrying two pods), left = Pod1, right = Pod2. See [Introduction to Biomek and Liquid Handling Method Writing](../Introduction-to-Biomek-Method-JSON.md#instrument-variants) §"Instrument Variants" / "Pod Types" for the instrument/pod matrix.

### DeckItems Value Structure

Each key in `deckItems` is a position name from the selected layout. To see the exact position names a given default deck defines (e.g. `P1`–`P12`, `TL1`, `WS1`), read that deck's sample under [`deck-layouts/samples/`](../biomek-file-formats/deck-layouts/samples/) — there is one `biomek-<model>-<deck>.deck.md` per deck. The value may be one of:

| Value Form | Meaning |
|------------|---------|
| `[]` (empty array) | Position is empty — no labware placed. |
| Array of labware objects | Labware stack placed at the position (bottom-up order). |
| String (e.g., `"Water"`) | For permanent-labware positions (wash stations): sets the liquid type on the existing labware. Does not replace labware. |

> **Position-name casing.** `deckItems` keys are the position name **camelCased** — only the first letter is lowercased; internal capitals are kept: `P1`→`p1`, `TR1`→`tR1`, `WS1`→`wS1`, `TL1`→`tL1`. Pipetting-step position VALUES, by contrast, use the display name (`P1`, `TR1`). Position matching is **case-insensitive**, so either form resolves — follow each step's own examples.

### Labware Object Structure

Each labware item in a `deckItems` array is a dictionary with:

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `_biomekType` | string | Yes | Always `"labware"`. Type discriminator read on import. |
| `class` | string | Yes | Labware class path (e.g., `"LabwareClasses\\BCFlat96"`, `"LabwareClasses\\BC230"`). |
| `volumeType` | string | No | Volume tracking mode for the labware. One of `"Unknown"`, `"Known"`, or `"Nominal"`. See §Enumerated / Constrained Values for meanings. Defaults to `"Unknown"` if omitted. |
| `properties` | object | No | Labware properties dictionary (liquid types, volumes, etc.). |
| `tipType` | string | No | For tip boxes only: the tip class the box holds, as a `TipClasses\\`-prefixed path (e.g., `"TipClasses\\T230"`) — the same path form `class` uses, and it must name a published tip class. This is **not** the same value as a Select Tips Load step's `tipType`, which takes a bare tip-*box* class name; see [Tip-class grammar](#tip-class-grammar) below. |
| `evalAmounts` | comArray object | No | Pre-evaluated well volumes or tip presence/usage information. See §`evalAmounts` / `evalLiquids` and §`evalAmounts` for tip boxes below. |
| `evalLiquids` | comArray object | No | Pre-evaluated liquid types per well. See §`evalAmounts` / `evalLiquids` below. |

#### Tip-class grammar

`tipType` appears in two places in method JSON and takes a **different value shape in each**:

| Where | What it names | Form | Example |
|-------|---------------|------|---------|
| A labware object in `deckItems` (this step) | The tip class **inside** the box | `TipClasses\` path | `"TipClasses\\T230"` |
| A [Multichannel Select Tips Load](32-Multichannel-Select-Tips-Load.md) / [Advanced Load](34-Multichannel-Select-Tips-Advanced-Load.md) step | The tip **box** labware class | Bare class name, no prefix | `"BC230"` |

Both must name a class that exists in the published catalogs — see [Catalog: Labware Classes](../biomek-file-formats/project-items/catalog-labware-classes.md) and [Catalog: Tip Classes](../biomek-file-formats/project-items/catalog-tip-classes.md). A free-text description such as `"230 uL tips"` raises an unknown-labware-class error, and so does putting the wrong one of the two forms in either place.

> **Tip-box to tip-class mapping.** A tip box's `tipType` is the tip class that corresponds to the box's labware class. The naming convention is that the tip class name matches the box's tip-size suffix: labware class `LabwareClasses\\BC230` (a 230 uL tip box) pairs with tip class `TipClasses\\T230`, and so on (`BC` prefix becomes `T`). For the complete box-to-tip-class pairing see the tip-box table in [Catalog: Labware Classes](../biomek-file-formats/project-items/catalog-labware-classes.md#tip-boxes); for what the `F`, `_LLS`, `_WB`, and `_384` suffixes mean see [Naming scheme](../biomek-file-formats/project-items/catalog-tip-classes.md#naming-scheme).

#### Labware `properties` sub-keys

The labware object's `properties` dictionary recognizes exactly these keys:

| Key | Type | Notes |
|-----|------|-------|
| `name` | string | Labware INSTANCE name (a user label; may be a Biomek expression). |
| `liquidtype` | string | A liquid-type NAME from the project (case-insensitive; may be an expression). Permanent labware (reservoirs / wash stations) typically set ONLY this. |
| `barcode` | string | Barcode text. |
| `senseEveryTime` | boolean | Advanced; only meaningful for liquid-level-sensing labware. Omit to default to first-time-only. |
| `device` | string | Advanced; only for labware whose class requires an associated device. |

`name`, `liquidtype`, and `barcode` are the common ones; `senseEveryTime` and `device` are advanced and rarely set. There is **no** `sealed` / `lid` key — lids are their own labware objects stacked in the `deckItems` array.

> **Tip-box routing keys are siblings of `properties`, not inside it.** On a tip-box labware object the routing keys `WhenDone`, `DiscardTips`, and `DiscardTipsLocation` are set at the labware object's TOP LEVEL (alongside `class`, `volumeType`, `properties`, `tipType`) — NOT inside `properties`:
> - `WhenDone` — string: `<Anywhere>` (default), `<Home>`, `<Any Trash>`, or a deck-position name. Not used on i3.
> - `DiscardTips` — boolean. The read default DIFFERS by pod (Span-8 defaults `true`, Multichannel defaults `false`), so set it explicitly when the behavior matters.
> - `DiscardTipsLocation` — string: `<TipBox>`, `<Any Trash>`, or a specific trash position (e.g. `TR1`). Apply to tip boxes. `<TipBox>` implies `DiscardTips: false`. **What it does depends on the pod family:**
> - **Multichannel and Fixed-8** — tips are returned to their own box, so `<TipBox>` is the value to use on a deck with no reachable trash.
> - **Span-8** — there is no return-to-box path. `<TipBox>` and `DiscardTips: false` do **not** keep the tips on the box; they route to a trash search exactly as `<Any Trash>` does, and fail identically when no trash position is both present and reachable by that pod. A Span-8 deck that unloads disposable tips needs a reachable trash position; no value of these keys substitutes for one.
>
> These three keys are a **separate vocabulary from the unload steps**. `"<where they came from>"` is a Fixed-8 Unload Tips `tipDestination` / Transfer `unloadLocation` value and is **not** accepted by `WhenDone` or `DiscardTipsLocation` — on those keys it is treated as a position name, resolves to nothing, and fails later. The `DiscardTipsLocation` equivalent is `<TipBox>`. See the tip-discard entry under [Common Mistakes](#common-mistakes).

> **Budget tip boxes to the method's tip usage.** Each Load-Tips step consumes a fresh set of tips from a tip box; once a box is emptied it is marked used and won't be re-picked. If a method's total Load-Tips cycles exceed the tips available across all placed tip boxes, enqueue fails with `Unable to find and retrieve tips of type <class> for <pod>` (naming the boxes already used). To run many tip-change cycles either (a) place additional tip boxes of the same class on free deck positions — but the deck has a limited number of positions, and a crowded deck of tall tip boxes can block a pod's motion path — or (b) keep the same tips loaded across consecutive steps that permit it, which is the simplest tip-saving pattern. Returning tips to their box with Unload Tips does **not** save tips by default: a tip-box well carries a use count of 1, so a tip that has pipetted is recorded as spent and a later Load Tips draws fresh tips instead. Re-picking used tips is possible with a higher use count, but treat it as advanced method programming — the software marks a tip used, and does not track which samples it has already touched, so keeping a reused tip away from unrelated samples is your accounting to do.

> **Labware instance names and how position fields resolve.** A labware INSTANCE name is set via the labware object's `properties.name`. Downstream, a pipetting step's position field (`where` / `location` / `position`, per step family) accepts ANY of: a literal deck-position name (e.g. `P5`), a labware instance name (the `properties.name` you assigned here), or a labware class name — all matched case-insensitively. If a name maps to several positions, the first is used. So e.g. `"location": "SourcePlate"` resolves when a labware placed here has `properties.name` set to `"SourcePlate"`. See [Concept Guide: Labware & Position Resolution](../concept-guides/02-labware-and-position-resolution.md#1-two-distinct-concepts) §1.

> **Waste position (for discarding bulk liquid).** Biomek has no "liquid waste" labware characteristic — the `Trash` characteristic (used by `<Any Trash>` and the tip-box `WhenDone` / `DiscardTipsLocation` routing keys) applies only to LABWARE and TIP discard, not to liquid, and the `Wash Station` characteristic serves only wash cycles. To dispose of bulk liquid, place an empty reservoir (or plate) on a free deck position here and name it (e.g. `"properties": {"name": "WastePlate"}`) — pipetting steps then discard by dispensing to that position as an ordinary destination. Give it capacity for the method's total discard volume; the per-well volume tracker will reject a dispense that exceeds the labware's `MaxVolume`. See the "Dispensing to waste" section in [Transfer](11-Transfer.md) for the authoring pattern and the pitfalls (do NOT dispense back into the source).

#### `evalAmounts` / `evalLiquids`

Both `evalAmounts` and `evalLiquids` are `comArray` objects with `arraySubtype: "variant"`. They pre-fill per-well state on the labware object — they are the starting balance of the enqueue-time volume ledger, not passive annotation. For what that ledger does downstream, see [Concept Guide: Well Volume Tracking](../concept-guides/05-well-volume-tracking.md).

- `evalAmounts` values are per-well volumes in µL, as numbers (or expression strings).
- `evalLiquids` values are per-well liquid-type NAME strings.

The array length must equal the labware's well count (96, 384, …). The array is **row-major**: index 0 = well A1, 1 = A2, … 11 = A12, 12 = B1, … 95 = H12 (for a 96-well plate). For a **reservoir** the addressable unit is the section, not the well, so a one-section reservoir takes a **one-element** array — passing 96 elements to it is the most common form of this mistake.

`evalAmounts` and `evalLiquids` react to a length mismatch **differently**, and the difference is a silent-corruption trap:

- **`evalAmounts` too *long*** fails at enqueue: `Position <Pn>: SetAllAmounts passed an array that was not the same size as the well count.` But **too *short* does not fail** — it silently pads the remaining wells with the array's **first** element's value (not `0.0`), so a partial array quietly seeds the wrong volumes with no error.
- **`evalLiquids`** rejects **any** length mismatch, too long or too short: `Position <Pn>: When using SetAll, the input array size must match the number of wells on the labware.`

Because the two are authored together, the only safe rule is a **full-length array for both**. There is no replication shortcut — a one-element array is correct only when the labware itself has one addressable unit (e.g., a one-section reservoir). Always supply a full-length array with explicit values for every well — use `0.0` for empty wells and `""` for wells with no assigned liquid.

Each value is also range-checked as it is seeded: a value above the well's (or reservoir section's) maximum fails with `Position <Pn>: Well volume out of allowed range: <value>.` Seed against the capacity in the per-type tables in [Catalog: Labware Classes](../biomek-file-formats/project-items/catalog-labware-classes.md), not against the nominal fill you have in mind.

They are consulted only when `volumeType` is `"Known"` or `"Nominal"`. If `volumeType` is `"Known"` but `evalAmounts` is omitted entirely, every well defaults to volume 0 ("known empty").

```jsonc
"evalAmounts": { "_biomekType": "comArray", "arraySubtype": "variant", "values": [200.0, 200.0, /* …96 total, row-major A1..H12… */] },
"evalLiquids": { "_biomekType": "comArray", "arraySubtype": "variant", "values": ["Water", "Water", /* …96 total… */] }
```

> **Seeding for downstream liquid-relative heights.** This step is where the tracked-volume surface for later liquid-relative moves comes from. On Fixed-8 and Multichannel (96- or 384-channel head) pods, declare the labware here with `volumeType` `"Nominal"` or `"Known"` and pair it with an `evalAmounts` fill (plus `evalLiquids`) — see the `volumeType` enum below and [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#technique-selection) §"Technique Selection" and §"Height Override" for the pod-family rules. Without `evalAmounts` the tracked volume stays 0 and the aspirate fails on available volume.

#### `evalAmounts` for tip boxes

For tip box labware classes, `evalAmounts` indicates tip presence and the number
of remaining uses of the tips. A value of `1` indicates that a tip is present
and may be used one time. A value of `0` indicates that no tip is present at
that location in the tip box.

The default behavior for a tip box is to set all values in `evalAmounts` to `1`,
indicating a full box of tips that can be used one time. To create a tip box
that is empty, set all values to `0`. To create a tip box that is partially
empty, set the pattern matching the well layout of the box to have `0` for empty
slots and `1` for occupied slots (ordering matches well ordering in microplates).

If the same tips are intended to be used repeatedly and contamination is not a
concern, for example to use the same tips to repeatedly mix a reservoir,
`evalAmounts` can be set to a higher number. It is recommended to name the tip
box in such situations so that the tips are **only** used for the expected purpose.
Higher numbers do not impact the Span-8 pod, as the Span-8 pod cannot unload
tips to a tip box.

Tip tracking is handled differently in Select Tips steps. Refer to the
individual steps for details.

Note that for Fixed-8 pods, tip usage is only decremented when tips contact
liquid. Loading and unloading without any liquid handling in between does not
decrement the usage. For Multichannel pods, tip usage is decremented each time
tips are loaded, regardless of if they are used.

## Enumerated / Constrained Values

### `volumeType`

| Value | Meaning |
|-------|---------|
| `"Unknown"` | Well volumes are not known, so a liquid-relative move has to locate the surface at run time. On **Span-8**, an LLS technique can sense the surface live even without a known volume. On **Fixed-8** and **Multichannel**, an `"Unknown"` well fails a liquid-relative move — those pods do not support liquid-level sensing (LLS); only Span-8 does. Declare at least `"Nominal"` there. This is the default if `volumeType` is omitted. See [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#liquid-relative-heights-need-a-locatable-liquid-surface) §"Liquid-relative heights need a locatable liquid surface". |
| `"Known"` | Well volumes are definitively known; pipetting steps skip LLS and calculate aspirate/dispense heights from tracked volumes. |
| `"Nominal"` | Well volumes are estimates. The estimate makes the surface locatable, so liquid-relative moves work the same as for `"Known"`. The difference is trust: a `"Known"` volume is taken as verified and used directly, while a `"Nominal"` well is refined by one live LLS sense before the move **on a pod that has LLS (Span-8) using a technique that senses** — after which that reading is reused for the rest of that well's operation. On **Fixed-8** and **Multichannel** there is no LLS, so `"Nominal"` uses the estimate directly, just like `"Known"`. Volume tracking keeps the uncertainty: an aspirate is refused only when it would drop the well below the low end of the estimate (`… has no more than <possible-max>.`), whereas a `"Known"` well is checked against its exact tracked amount. |

In JSON the single `volumeType` string is authoritative. The parser is case-insensitive: `"Known"`, `"KNOWN"`, and `"known"` are all accepted. Numeric strings (e.g. `"3"`) are explicitly rejected even if they correspond to the enum's underlying integer value; any other non-matching value also fails to import.

> **Destination volume auto-tracking.** The engine automatically updates tracked well volumes and liquid types on destination labware after a dispense, regardless of that labware's initial `volumeType`. A plate placed here as `"Unknown"` will have per-well volumes tracked once liquid has been dispensed into it, so a downstream Aspirate or Mix step using `"Well Contents"` volume on that plate works correctly.

> **Destination labware must have DEFINED amounts before the first liquid-relative dispense.** This is distinct from the auto-tracking note above, which updates volumes *after* a dispense completes. Most default pipetting techniques (including `"F8 Medium"`, `"S8 1000 Medium"`, `"MC P60"`) position the tips relative to the liquid level, and that move requires a locatable volume surface even at the destination -- before any liquid has been dispensed into it. If a destination plate has `volumeType: "Unknown"`, the first Dispense step targeting it fails with: `Cannot move relative to liquid level if liquid amounts are not defined for specified labware.` To avoid this, declare the destination `"Known"` or `"Nominal"` with an `evalAmounts` array (all `0.0` for an initially empty plate) and an `evalLiquids` array. This applies to Fixed-8, Multichannel, and Span-8 destinations alike. See the seeding format under [evalAmounts / evalLiquids](#evalamounts--evalliquids) and the [Worked Example](../Worked-Example-End-to-End.md) for a complete method with a correctly seeded destination.

> **Volume tracking (running per-well ledger).** Once a labware is placed with `"Known"` or `"Nominal"` (or has been dispensed into and auto-tracked as noted above), the enqueue engine maintains a running per-well volume across every downstream pipetting step. Two hard-fail cases follow:
>
> - **Dispense exceeds the well's capacity.** If a dispense would push the well past the labware class's max well volume, enqueue fails with `Cannot dispense <amt> because the <well> already has <current> and can only hold <max>.` The `<max>` is the class's per-well capacity — see the max-volume column in the per-type tables under [Catalog: Labware Classes](../biomek-file-formats/project-items/catalog-labware-classes.md) (per-class; on a reservoir it is the per-section max volume).
> - **Aspirate exceeds the well's tracked volume.** If a well is tracked as `"Known"` (or an aspirate would drop a `"Nominal"` well below zero considering the low end of its uncertainty), enqueue fails with `Cannot aspirate <amt> because the <well> has only <current>.` (or `… has no more than <possible-max>.` for `"Nominal"`).
>
> Keep the seed here (`volumeType` + `evalAmounts`) consistent with the volumes each downstream aspirate/dispense will actually move — the tracker propagates through Transfers, Combines, Mix, Serial Dilution, etc.

### `leftHasTips` / `rightHasTips` (tri-state)

| JSON Value | Behavior |
|------------|----------|
| `true` | Pod is configured with tips of the type named in the corresponding `*TipType` key. |
| `false` | Pod is set to have no tips loaded. |
| `null` | Pod tip status is not changed; confirmation dialog reports current state. |

### `deckItems` position value sentinel

| JSON Value | Behavior at Enqueue |
|------------|---------------------|
| `[]` (empty array) | Position cleared of labware (nothing placed). |
| String value | Permanent-labware position: sets `LiquidType` property and `Liquid` dataset on existing labware. |

### `layout`

`layout` chooses which of the instrument's decks the step configures. Deck names are per-instrument
configuration — the deck one instrument calls `hybrid` may not exist on another — so **no literal
is safe to copy from an example**.

**Set `layout` explicitly — do not omit it.** Omitting the key falls back to the instrument's
default deck, so a method later run on an instrument whose default deck differs from the one it was
written for can behave unexpectedly — and silently, because omission never fails at enqueue. Pinning
the deck makes a wrong deck either fail loudly (an unknown name is rejected) or run correctly (the
right name resolves). How to do it:

- **Set it to a deck name read from that instrument's settings export.** Matching is
  case-insensitive. Secondary caveat: a method that pins a deck is not portable to an instrument configured differently — but that portability cost buys the loud-failure protection above.
- **Do not guess a name.** An unknown deck fails at enqueue and the message names the deck rather than the mistake.

**Omitting `layout` is a discouraged fallback.** The step then uses the instrument's default deck and
cannot name a deck that does not exist — but the choice is not neutral: it goes to whichever deck is
flagged default, or, if none is, to an arbitrary one. That unpredictability across instruments is exactly why setting `layout` explicitly is recommended.

If the deck you name is not the one currently loaded, the step proceeds and warns:

```
WARNING: This method was configured with <layout> but <current deck> is the current deck.
```

Deck names are set at instrument creation and can be renamed, added, or deleted in the Deck Editor. To see the deck names an instrument defines and its default, read its Instrument Settings JSON export (see [Deck Layouts Format Specification](../biomek-file-formats/deck-layouts/deck-layouts-format-spec.md#finding-and-using-the-instruments-default-deck) §"Finding and using the instrument's default deck"); produce one from the Biomek software if you don't have it.

> **`layout` names a deck, not a pod family.** `"Fixed8"` looks like a `podType` token but is not a
> deck name on any factory-configured instrument — the Fixed-8 (i3) deck is exported as `"i3"`.
> `"layout": "Fixed8"` fails the enqueue check. Some factory deck names do coincide with pod-type tokens (`"Span8"` and `"Multichannel"` are real decks on the instruments configured with them), which is what makes the mistake easy.

## Cross-Field Validation Rules

1. When `layout` is present, it must name a deck defined on the instrument; otherwise enqueue fails. When `layout` is absent (the normal form), the instrument's default deck is used. If the resolved value is an empty string (no default configured and no key provided), enqueue fails.
2. Each key in `deckItems` should correspond to a position in the selected layout. Keys that do not match a layout position are silently ignored (not placed).
3. When `leftHasTips` / `rightHasTips` is `true`, the corresponding `*TipType` must name a valid tip box class in the instrument's labware-class library. An invalid name causes a tip-box-not-found failure.
4. Any labware `class` referenced in `deckItems` must exist in the instrument's labware-class library; otherwise that labware is not placed and a mapping error is shown.
5. A string value in `deckItems` (liquid-type setting) only takes effect if the position already has permanent labware. If the position has no labware, the liquid type is not applied.
6. When `verifyDeckOnly` is `true`, labware placement and pod setup are skipped entirely — only pause/barcode behavior runs.
7. This step is where a deck position gets its labware `class`, its `volumeType`, and any `evalAmounts` / `evalLiquids` fill; pipetting steps do not create labware and inherit that seeding. If a downstream step uses a liquid-relative height, seed it here — see the note under §Labware Object Structure, [Concept Guide: Labware & Position Resolution](../concept-guides/02-labware-and-position-resolution.md#4-labware-must-be-declared-before-it-is-used) §4, and [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#technique-selection) §"Technique Selection" and §"Height Override".

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

### Example 1: Minimal Instrument Setup (empty deck, no pause)

```json
{
  "stepType": "Instrument Setup",
  "parameters": {
    "barcodeInput?": false,
    "deckItems": {
      "p1": [],
      "p2": [],
      "p3": [],
      "p4": [],
      "p5": [],
      "p6": [],
      "p7": [],
      "p8": [],
      "p9": [],
      "p10": [],
      "p11": [],
      "p12": []
    },
    "layout": "i3",
    "pause?": false,
    "podSetup": {},
    "verifyPodSetup?": false
  }
}
```

### Example 2: Typical setup with tip box and plate, pod verification enabled

```json
{
  "stepType": "Instrument Setup",
  "parameters": {
    "barcodeInput?": false,
    "deckItems": {
      "p1": [],
      "p2": [],
      "p3": [],
      "p4": [],
      "p5": [],
      "p6": [],
      "p7": [
        {
          "_biomekType": "labware",
          "class": "LabwareClasses\\BC230",
          "volumeType": "Unknown",
          "properties": {},
          "tipType": "TipClasses\\T230"
        }
      ],
      "p8": [],
      "p9": [
        {
          "_biomekType": "labware",
          "class": "LabwareClasses\\BCFlat96",
          "volumeType": "Unknown",
          "properties": {}
        }
      ],
      "p10": [],
      "p11": [],
      "p12": []
    },
    "layout": "i3",
    "pause?": true,
    "podSetup": {
      "leftHasTips": false,
      "leftTipType": "",
      "rightHasTips": false,
      "rightTipType": ""
    },
    "verifyPodSetup?": true
  }
}
```

> The p9 plate here is placed `"Unknown"` (no tracked volumes). To seed known well volumes instead, set `volumeType` to `"Known"` (or `"Nominal"`) and add matching `evalAmounts` / `evalLiquids` arrays on that labware object — see §`evalAmounts` / `evalLiquids`. A `"Known"` plate without `evalAmounts` treats every well as volume 0 (per Cross-Field Validation rule 7 and the seeding note).

### Example 3: Setup with pod tip configuration and wash station liquid type

```json
{
  "stepType": "Instrument Setup",
  "parameters": {
    "barcodeInput?": false,
    "deckItems": {
      "p1": [],
      "p2": [],
      "p3": [],
      "p4": [],
      "p5": [],
      "p6": [],
      "p7": [
        {
          "_biomekType": "labware",
          "class": "LabwareClasses\\BC230",
          "volumeType": "Unknown",
          "properties": {},
          "tipType": "TipClasses\\T230"
        }
      ],
      "p8": [
        {
          "_biomekType": "labware",
          "class": "LabwareClasses\\BCFlat96",
          "volumeType": "Known",
          "properties": {}
        }
      ],
      "p9": [],
      "p10": [],
      "p11": [],
      "p12": [],
      "tL1": [],
      "tL2": [],
      "tR1": [],
      "wS1": "Water"
    },
    "layout": "Multichannel",
    "pause?": true,
    "podSetup": {
      "leftHasTips": true,
      "leftTipType": "BC230",
      "rightHasTips": null,
      "rightTipType": ""
    },
    "verifyPodSetup?": true
  }
}
```

### Example 4: i5 Span-8 deck with tip box and plate

```json
{
  "stepType": "Instrument Setup",
  "parameters": {
    "barcodeInput?": false,
    "deckItems": {
      "p1": [
        {
          "_biomekType": "labware",
          "class": "LabwareClasses\\BC230",
          "volumeType": "Unknown",
          "properties": {},
          "tipType": "TipClasses\\T230"
        }
      ],
      "p2": [],
      "p3": [],
      "p4": [],
      "p5": [],
      "p6": [
        {
          "_biomekType": "labware",
          "class": "LabwareClasses\\BCFlat96",
          "volumeType": "Unknown",
          "properties": {
            "name": "SamplePlate"
          }
        }
      ],
      "p7": [],
      "p8": [],
      "p9": [],
      "p10": [],
      "p11": [],
      "p12": [],
      "p13": [],
      "p14": [],
      "p15": [],
      "p16": [],
      "p17": [],
      "p18": [],
      "p19": [],
      "p20": []
    },
    "layout": "Span8",
    "pause?": true,
    "podSetup": {
      "leftHasTips": false,
      "leftTipType": "",
      "rightHasTips": false,
      "rightTipType": ""
    },
    "verifyPodSetup?": true
  }
}
```

### Example 5: i5 Multichannel deck with tip-loader, plates, and wash station

```json
{
  "stepType": "Instrument Setup",
  "parameters": {
    "barcodeInput?": false,
    "deckItems": {
      "tL1": [
        {
          "_biomekType": "labware",
          "class": "LabwareClasses\\BC230",
          "volumeType": "Unknown",
          "properties": {},
          "tipType": "TipClasses\\T230"
        }
      ],
      "tL2": [],
      "tL3": [],
      "tL4": [],
      "tL5": [],
      "p1": [
        {
          "_biomekType": "labware",
          "class": "LabwareClasses\\BCFlat96",
          "volumeType": "Known",
          "properties": {
            "name": "SourcePlate"
          }
        }
      ],
      "p2": [],
      "p3": [],
      "p4": [],
      "p5": [],
      "p6": [
        {
          "_biomekType": "labware",
          "class": "LabwareClasses\\BCFlat96",
          "volumeType": "Unknown",
          "properties": {
            "name": "DestPlate"
          }
        }
      ],
      "p7": [],
      "p8": [],
      "p9": [],
      "p10": [],
      "p11": [],
      "p12": [],
      "p13": [],
      "p14": [],
      "p15": [],
      "wS1": "Water"
    },
    "layout": "Multichannel",
    "pause?": true,
    "podSetup": {
      "leftHasTips": false,
      "leftTipType": "",
      "rightHasTips": false,
      "rightTipType": ""
    },
    "verifyPodSetup?": true
  }
}
```

> **Source-plate volume type on Multichannel.** The source plate at `p1` uses `volumeType: "Known"` because Multichannel pods cannot perform a liquid-relative move on an `"Unknown"` well -- Multichannel pods do not support liquid-level sensing (LLS); only Span-8 does (see §`volumeType`). For a source you aspirate from with a liquid-following technique, use `"Known"` or `"Nominal"` and pair it with `evalAmounts` to seed per-well volumes -- see §`evalAmounts` / `evalLiquids`. A `"Known"` plate without `evalAmounts` treats every well as volume 0.

## Common Mistakes

- **Bare labware class name without the `LabwareClasses\\` prefix**: Instrument Setup's `deckItems.*[].class` needs the fully-qualified path (e.g. `"LabwareClasses\\BCFlat96"`) — a bare `"BCFlat96"` fails at enqueue with a load-labware error at that position. The prefix requirement is specific to this step: other steps (Transfer/Combine/MC Select Tips, etc.) reference the same labware class by its short name (`BCFlat96`).
- **Changing a labware class without propagating it**: A class edit here must be mirrored in **every** downstream step referencing that position or class — the `labwareClass` / `what` / tip-box / plate-class fields in Transfer, Combine, Aspirate/Dispense/Mix, Move Labware, Hold Labware, etc. A stale reference fails at enqueue with a class-mismatch error. Treat a setup-class change as a method-wide find-and-update.
- **Reversed labware stacking order**: Items in a `deckItems` array are bottom-up — the *last* array element is the topmost (accessible) labware. Getting the order wrong at a lidded/nested position places the wrong labware on top.
- **Confusing tip type with tip class**: `leftTipType` / `rightTipType` name a tip *box* labware class as the **bare** class name (e.g., `"BC230"`) — the same form a Select Tips step uses — NOT a `LabwareClasses\\`-prefixed path and NOT a raw tip class like `"TipClasses\\T230"`. A prefixed value like `"LabwareClasses\\BC230"` is unbound and fails at enqueue with `the tip box type LabwareClasses\BC230 not found.`
- **String value on a non-permanent position**: A string in `deckItems` sets liquid type on existing permanent labware (wash stations only). At a regular deck position it is silently ignored — the position stays empty and no error surfaces.
- **Omitting `_biomekType` on a labware object**: Without `"_biomekType": "labware"` Biomek will not recognize the item as labware on import — it drops out of the deck silently.
- **Multichannel tip box on a `p*` position instead of `tL*`**: On a Multichannel deck, a tip box used for MC tip loading must be placed on a tip-loader position (`tL1`--`tL5`, which carry the `"multichannel Can Load Tips"` characteristic). Placing it on a `p*` position causes a runtime "unable to find/retrieve tips" failure. Span-8 tip loading similarly requires positions with the `"span Can Load Tips"` characteristic. Example 5 shows the correct Multichannel placement (tip box on `tL1`).
- **`"<Any Trash>"` on a deck with no reachable trash**: `"<Any Trash>"` resolves only to a deck position carrying the tip-discard characteristic, and "present on the deck" is not the same as "reachable by the pod that must use it" — on a two-pod instrument each pod may reach only some of the trash positions, or none. A deck that defines no trash position at all fails outright. The replacement depends on **which key** you are setting, and the two families do not share a vocabulary:
  - Fixed-8 Unload Tips `tipDestination`, and Transfer / Combine `unloadLocation`: use `"<where they came from>"` (return tips to their originating box).
  - The tip-box routing keys `WhenDone` and `DiscardTipsLocation`: `"<where they came from>"` is **not** a valid value for either — set `DiscardTipsLocation` to `"<TipBox>"` (which also means `DiscardTips` is `false`), and leave `WhenDone` at `"<Anywhere>"`. **This remedy is Multichannel/Fixed-8 only.** A Span-8 pod has no return-to-box path, so `"<TipBox>"` still routes to a trash search and still fails on a deck with no reachable trash — see the `DiscardTipsLocation` entry above. On Span-8 the only fix is a trash position the pod can actually reach.

  Using `"<where they came from>"` on `DiscardTipsLocation` does not fail validation; it is read as an ordinary position name, fails to resolve, and surfaces later as a tip-discard-location error that names the pod rather than the key.

- **Destination plate left `"Unknown"` when the technique follows liquid level**: Most default pipetting techniques move relative to the liquid surface. If a destination plate is placed with `volumeType: "Unknown"` (the default when `volumeType` is omitted), the first Dispense step targeting it fails with `Cannot move relative to liquid level if liquid amounts are not defined for specified labware.` This is NOT rescued by auto-tracking (auto-tracking only runs after a dispense succeeds). Fix: set the destination to `volumeType: "Known"` (or `"Nominal"`) with `evalAmounts` all `0.0` and `evalLiquids` filled. This applies to Fixed-8, Multichannel, and Span-8 destinations.
