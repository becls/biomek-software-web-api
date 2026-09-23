# Concept Guide: Labware & Position Resolution

Almost every physical step names *where* it acts (a **deck position**) and *what kind of
labware* is there (a **labware class**). These are two different things, and the **JSON key
names for them vary by step family**. This guide pins down the distinction and the
per-family key spellings so you address the right thing with the right key.

---

## 1. Two distinct concepts

- **Deck position** — a named slot on the instrument's deck (e.g. `"P9"`, `"P13"`, a wash
  station like `"WS1"`, or a trash like `"TR1"`). It identifies *a physical location*.
  Position names come from the target instrument's deck layout (`P1`…`P30`, `WS1`, `TR1`,
  `TR2`, `W1`, tip-load positions, …); do not invent them.
- **Labware class** — the *type/geometry* of the container that sits there (e.g.
  `"BCFlat96"`, `"CostarFlat384Square"`). A class defines well count, spacing, depth, per-well capacity (maximum volume), and dead
  volume. It does **not** say where the labware is.

A step generally needs **both**: the position tells the pod where to go; the class tells it
the well geometry to pipette into. At enqueue the software checks that the class you named
matches the labware actually placed at that position — a mismatch fails with a
class-mismatch error.

A position field resolves its value **case-insensitively** as **either** a literal
deck-position name (`"P13"`), a labware **instance** name (the labware's `properties.name`),
**or** a labware **class** name — if a name maps to several positions, the first is used.
All three forms resolve to the same physical labware. Not all steps support all
options. Refer to the individual step's documentation.

Prefer using labware **instance names** when possible. If the instance name is
used, then changes to the labware's position on deck in the Instrument Setup
step do not require updates to downstream steps.

> **Note — position-name casing.** The **keys** under `deckItems` and the deck-layout's
> `positions` object are the position name **camelCased** — only the first letter is
> lowercased; internal capitals are kept: `P1` → `p1`, `TR1` → `tR1`, `WS1` → `wS1`,
> `TL1` → `tL1`. Pipetting-step position **values** (`where`, `location`, `position`) use
> the display form (`P1`, `TR1`, …). Because position matching is case-insensitive, either
> form resolves — follow each step's own examples.

## 2. Key-name variance by step family

The serialized key names are **not** uniform. Use the spelling the target step family
expects:

| Step family | Position key | Labware-class key |
|-------------|--------------|-------------------|
| **Span-8 / Fixed-8** (Aspirate, Dispense, Mix, …) | `where` | `what` |
| **Multichannel** (Aspirate, Dispense, Mix, …) | `location` | `labwareClass` |
| **Transfer / Combine** *(per item in `items`)* | `position` | `labwareClass` |
| **Instrument Setup** *(per labware in `deckItems`)* | the `deckItems` key **is** the deck position | `class` (on the labware object) — path-prefixed, see note below |

So "the source plate class" is `what` on a Span-8 or Fixed-8 step, `labwareClass` on a
Multichannel or Transfer-item, and `class` on an Instrument Setup labware object — the *same
concept*, three key spellings. Author whichever the step's own parameter table lists; do not
carry a Span-8/Fixed-8 `what`/`where` into a Multichannel step.

> **Note — `class` value form.** Mixing the two class-name forms is a common hard enqueue failure: using the `LabwareClasses\\`-prefixed form in a pipetting step, or the bare short name in `deckItems`, both fail. The rule: on Instrument Setup labware objects, `class` is serialized **path-prefixed** — e.g. `"LabwareClasses\\BCFlat96"`; every pipetting step's `what` / `labwareClass` field takes the **short** class name only (e.g. `"BCFlat96"`). The `LabwareClasses\\` prefix belongs **only** in Instrument Setup's `deckItems` `class` field. Author each key in the form its own step's parameter table shows.

## 3. Positions can be expressions

On the pipetting steps the position field is expression-capable: Span-8 `where` and
Multichannel `location` both take either a literal deck position (`"P13"`) or a `=`-prefixed
expression that resolves to one at run time (e.g. `"=sourcePos"`). A Transfer/Combine item's
`position` takes a literal deck position or labware name — check the step's own parameter
table before assuming expression support. The labware-class field is normally a literal class
name. See [Concept Guide: Expressions](01-expressions.md).

## 4. Labware must be declared before it is used

A pipetting step does not *create* labware — it references labware already placed on the
deck by an **Instrument Setup** step earlier in the method. That setup
step is where a position gets its labware class, its `volumeType`, and any `evalAmounts` /
`evalLiquids` fill (see [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#liquid-types) §"Liquid Types" and §"Volume types and locatable surfaces"). Both `evalAmounts` and `evalLiquids` arrays must be full-length — exactly equal to the labware's well count (or section count for a reservoir); see [Instrument Setup](../step-documentation/03-Instrument-Setup.md#evalamounts-evalliquids) §`evalAmounts` / `evalLiquids`. Downstream steps must name
a class **consistent with** what setup placed at that position.

**Consequence for edits:** if you change the labware class at a position in Instrument
Setup, every downstream field that names that class (`what` / `labwareClass`, and tip-box
class references such as a Load-Tips step's `tips` *when it holds a class* or a Select-Tips
Load's `tipType`) must be updated to match, or the step fails at enqueue. A Select Tips Load's `tipsLocation`
is a *deck-position* reference, not a class name — it needs updating only if the tip box is
moved to a different position, not when its class changes.

## 5. Tips are labware too

Tip boxes are labware on the deck (e.g. class `"BC230"`; see
[Catalog: Labware Classes](../biomek-file-formats/project-items/catalog-labware-classes.md)),
and tip-loading steps address them — but the **key names differ from the
pipetting steps above**, and the plain Load-Tips steps have no separate position
key.

Multichannel Load Tips, Span-8 Load Tips, and Fixed-8 Load Tips all use the
`tips` key only, which may be the box class, box name, **or** deck position.

Only Select Tips Load splits class (`tipType`) from position (`tipsLocation`). On that step
`tipType` is a **bare** tip-box class name (`"BC230"`) — do not prefix it. The identically-named
key on an Instrument Setup labware object is a different value: the tip class *inside* the box,
written as a `TipClasses\` path (`"TipClasses\\T230"`). See
[Tip-class grammar](../step-documentation/03-Instrument-Setup.md#tip-class-grammar).

So the "tip class to load" is `tips` on a Multichannel/Span-8/Fixed-8 Load Tips step but
`tipType` on a Select Tips Load step; an explicit tip-box position exists only as
`tipsLocation` (Select Tips Load) or, on any Load Tips step, as a value passed through
`tips`. A named
tip class or position that does not resolve to an available box on the deck fails.

Author the key
the target step's own parameter table lists — do not invent a tip-source key on a Load
Tips step (its tips come from `tips` / `tipType`). Note `tipLabwareClass` is a real key
on the lower-level pipetting steps (Fixed-8 / Span-8 / Multichannel Aspirate, Dispense,
Mix — paired with `refreshTips` — and on Hold Labware), so this warning is about not
copying it onto steps that don't list it.

## 6. Reachability

The auto-locate above ("a **reachable** box") uses the same envelope constraint that
governs every pipetting step. Each pod can only service deck positions within its
physical travel envelope, and on a multi-pod instrument the two pods' envelopes overlap
only in part. A step that names a `position` (or `where` / `location`, per family)
**outside the chosen pod's reach** fails at enqueue or run even though the position and
labware class resolve fine. When you switch a method from one pod to another (e.g. MC →
Span-8), re-check every position field — layouts valid for one pod are not automatically
reachable by the other. To find which positions a pod can reach, consult the target
instrument's deck configuration rather than assuming; the Biomek Editor's own deck view
is the authoritative source.

A tip trash resolves the same way, and is a common surprise: tip-unload steps name neither a
labware class nor a device, but search for a position carrying the `can Discard Tips`
characteristic — on Span-8 and Multichannel, one the unloading pod can reach. The three failure
messages and the per-family remedies are in
[Reachability and Access](../biomek-file-formats/deck-layouts/reachability-and-access.md)
§"Symptom to likely cause".

## 7. Quick checklist

- Am I using the position/class **key names for this step family** (`what`/`where` vs
  `labwareClass`/`location` vs `labwareClass`/`position` vs `class`)?
- Does the class I name match what Instrument Setup placed at that position?
- Are deck-position names real ones from the target deck layout, not invented?
- After a labware-class change in setup, did I update every downstream class reference and
  tip-box reference?
- If the method unloads tips, does the target deck carry a `Can Discard Tips` position that
  the unloading pod can reach? (Check per pod on a two-pod instrument.)
