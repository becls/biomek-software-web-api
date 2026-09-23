# Biomek Method JSON — Quick-Reference Cheatsheet

One-page summary of the rules that bite most often when authoring method JSON by hand.
Each section links to the full reference for depth.

> **Want a working template?** See [Worked Example: Simple i3 Fixed-8 Method](Worked-Example-End-to-End.md) for a complete, copy-ready i3/Fixed-8 method you can modify.

---

## Method Envelope

([Method JSON Structure](Method-JSON-Structure.md))

```jsonc
{
  "format": "Biomek Method",       // required, must equal "Biomek Method" (compared case-insensitively)
  "formatVersion": "1.0",          // required; the Biomek Method JSON format version, currently "1.0"
  "author": "",                    // optional
  "description": "",               // optional
  "steps": [ ... ]                 // required, the method tree
}
```

---

## Step Node Shape

([Method JSON Structure](Method-JSON-Structure.md))

```jsonc
{
  "stepType": "Transfer",          // required -- case-insensitive match
  "disabled": false,               // optional (omitted when false)
  "parameters": { ... },           // required (may be {})
  "subSteps": [ ... ]              // optional -- containers only
}
```

**`disabled`**: Author `"disabled": true` on the step node to disable it without removing it from the method. ([Base Keys & Serialization Conventions](step-documentation/00-Base-Keys-and-Conventions.md#3-disabling-a-step-disabled) §3)

---

## Anchor & Terminator Rules

([Method JSON Structure](Method-JSON-Structure.md))

| Rule | Detail |
|------|--------|
| Start/Finish required at root | Every method root: `[Start, ...body..., Finish]`. |
| Free-form containers need a terminator | Last child must be `"End"` (most containers), `"Multichannel Select Tips End"` (Select Tips family), or the terminator an add-on container registers for itself. |
| Anchors can never be disabled | `disabled: true` on Start, Finish, or any terminator (End, Multichannel Select Tips End, etc.) is rejected at validation. |
| Exact-children: If | Exactly 2 `"Named Container"` children (Then / Else), each ending with `"End"`. |

---

## Common `_biomekType` Discriminators

([Method JSON Structure](Method-JSON-Structure.md))

| `_biomekType` | Minimal shape |
|---|---|
| `"comArray"` | `{ "_biomekType": "comArray", "arraySubtype": "<type>", "values": [...] }` |
| `"labware"` | `{ "_biomekType": "labware", ... }` (inside Instrument Setup) |
| `"dateTime"` | `{ "_biomekType": "dateTime", "value": "2024-01-31T13:45:30.0000000Z" }` |
| `"technique"` | `{ "_biomekType": "technique", ... }` (inline custom technique — usually editor-written, but valid in JSON; overrides `prototype`) |
| *(absent)* | Plain nested dictionary (the default) |

Never use `_biomekType` as your own key inside `parameters`.

### `comArray` One-Liner

```jsonc
{ "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true, false, ...] }
```

Legal `arraySubtype` values: `boolean`, `byte`, `dateTime`, `double`, `integer`, `short`, `single`, `string`, `variant`.

A plain JSON array (`[true, false]`) without the wrapper is deserialized as a different Biomek type and will break downstream logic.

---

## Pod-Family --> Step-Family

([Introduction to Biomek and Liquid Handling Method Writing](Introduction-to-Biomek-Method-JSON.md))

| Pod | Steps |
|-----|-------|
| Fixed-8 (i3) | `Fixed-8 Aspirate`, `Fixed-8 Dispense`, `Fixed-8 Mix`, `Fixed-8 Load/Unload Tips` |
| Span-8 | `Span-8 Aspirate/Dispense/Load/Unload/Wash Tips`, `Span-8 Serial Dilution`, `Span-8 Transfer From File` |
| Multichannel | `Multichannel Aspirate/Dispense/Mix/Load/Unload/Wash Tips`, `Multichannel Select Tips *` |
| Any | `Transfer` (pod family resolved from the `pod` field at runtime) |
| Span-8 or Multichannel | `Combine` (not available on a Fixed-8 pod) |

Low-level pipetting steps are pod-prefixed (`Fixed-8`, `Span-8`, `Multichannel`); `Transfer` and `Combine` are not — they pick their pod family at run time from the `pod` field.

---

## Expression `=` Prefix

([Concept Guide: Expressions](concept-guides/01-expressions.md))

A string whose first character is `=` is evaluated at run time using VBScript syntax; without it the value is literal.

```jsonc
"amount": "10"         // literal 10 uL
"amount": "=dose * 2"  // expression, evaluated at run time
```

Write numeric values destined for expression-capable fields as **quoted strings** (`"10"`, not `10`). Some fields require an explicit flag (`useWellExpression`, `useExpression`) before the expression key is read — setting the expression without flipping the flag has no effect; the engine reads the literal key instead.

**Exception:** condition fields (e.g. `If.condition`) auto-prefix the `=`, so a bare string there is still evaluated as an expression. ([Concept Guide: Expressions](concept-guides/01-expressions.md#1-the-prefix) §1)

---

## Well-Selection Idioms

([Introduction to Pipetting Steps](Introduction-to-Pipetting-Steps.md#well-numbering) §"Well Numbering"; [Concept Guide: Probe & Mandrel Selection](concept-guides/03-probe-and-mandrel-selection.md))

| Idiom | Where | Shape |
|-------|-------|-------|
| **firstWell + stride** | Span-8 / Fixed-8 low-level steps | `"firstWell": 1` (1-based, row-major). Pod adds `+WellsX` per tip down the column for 96-format plates. |
| **pattern comArray** | Transfer / Combine items, Span-8 Serial Dilution | `"pattern": { "_biomekType": "comArray", "arraySubtype": "boolean", "values": [true, false, ...] }` — one entry per addressable position, full-length: 96 booleans for a 96-well plate, but one per **section** on a reservoir (`[true]` for a one-section reservoir). **Which of `pattern` and `selectionInfo` is the operative well selector depends on pod family** — Fixed-8 and Multichannel read `selectionInfo`; Span-8 reads `pattern`. See Rule 12 in [Transfer](step-documentation/11-Transfer.md). Keep both keys present and consistent with each other on Transfer/Combine items. |
| **selectionInfo sections** | Multichannel Aspirate / Dispense / Mix | `"selectionInfo": { "_biomekType": "comArray", "arraySubtype": "integer", "values": [1] }` — 1-based section/quadrant index. 96-well + 96-head = section `[1]` only; 384-well + 96-head = quadrants `1`--`4`. |

### The `=Col` vs `=1+(Col-1)*8` Stride Trap

To sweep plate columns 1..12 with a column-spanning pod, the first-well expression is **`"=Col"`** (the column index itself). Do **not** write `"=1+(Col-1)*8"` — that assumes column-major numbering, lands the first tip mid-plate on a row-major layout, and fails at enqueue because the pod cannot fit its remaining tips.

---

## Fixed-8 Never Auto-Selects a Technique

([Pipetting Techniques, Templates, and Auto-Selection](Pipetting-Techniques-and-Templates.md))

Setting `autoSelectPrototype: true` on a Fixed-8 Aspirate / Dispense / Mix step throws at enqueue. This is permanent and by design. Always use `autoSelectPrototype: false` with an explicit `prototype` (e.g. `"F8 Medium"`) on Fixed-8 steps.

---

## Start & Finish Steps Special Import Requirements

([Start](step-documentation/01-Start.md), [Finish](step-documentation/02-Finish.md))

| Step | Must author |
|------|-------------|
| **Start** | `"bitmap": "OStepUI.ocx,START"`, plus `"let": {}`, `"weak": {}`, `"prompt": {}` (empty is fine). Omitting any of these fails at enqueue or breaks the icon. |
| **Finish** | `"bitmap": "OStepUI.ocx,FINISH"`. Omission breaks the icon. |

Authored `parameters` replace defaults wholesale — nothing is merged in.

---

## Key Casing

([Base Keys & Serialization Conventions](step-documentation/00-Base-Keys-and-Conventions.md#1-key-name-casing-spelling-global-rule) §1)

Keys are **case-insensitive on import**, but author the **serialized (camelCase-first-word) form** for round-trip fidelity. The rule: only the first PascalCase word is lowercased; everything else (spaces, underscores, trailing `?`) is preserved verbatim.

| Source constant | Serialized JSON key |
|---|---|
| `LiquidType` | `liquidType` |
| `Use Shake` | `use Shake` |
| `HTML_Dialog` | `htmL_Dialog` |
| `Dynamic?` | `dynamic?` |

---

## Key Gotchas

| Gotcha | Fix |
|--------|-----|
| **Span-8 has no standalone Mix step** | Mixing in place is a technique setting — configure it on the technique. An item-level `mixCount` on a Transfer item is inert on every pod family. ([Transfer](step-documentation/11-Transfer.md), [Pipetting Techniques, Templates, and Auto-Selection](Pipetting-Techniques-and-Templates.md)) |
| **MC partial-plate needs Select Tips** | A `Multichannel Aspirate`/`Dispense` alone cannot pick an arbitrary well subset. Wrap in `Multichannel Select Tips` container (terminator: `Multichannel Select Tips End`). ([Introduction to Biomek and Liquid Handling Method Writing](Introduction-to-Biomek-Method-JSON.md)) |
| **A technique is built for one pod family** | A technique built for one pod family won't apply to another — referencing it from the wrong pod fails or mis-renders. The **default** techniques signal their family in their names (`F8 …` = Fixed-8, `S8 …` = Span-8, `MC …` = Multichannel), but a technique you create can be named anything, so match by the pod family a technique was built for, not by its name. ([Pipetting Techniques, Templates, and Auto-Selection](Pipetting-Techniques-and-Templates.md)) |
| **`LabwareClasses\\` prefix only in Instrument Setup** | In `deckItems`, the class field takes the path-prefixed form (e.g. `"class": "LabwareClasses\\BCFlat96"`). In pipetting steps' `what` / `labwareClass` fields, use the **short name** only (`"BCFlat96"`). Using the prefixed form in a pipetting step causes an enqueue failure. |
| **Budget tip boxes to tip usage** | Too many Load-Tips cycles for the placed boxes fails enqueue (`Unable to find and retrieve tips…`); add boxes, or keep the same tips loaded across consecutive steps. |
| **Not every deck position is reachable by every pod** | Pipetting at an unreachable position fails enqueue (`Unable to find a path …`). If the message continues `destination <axis> … outside of travel range`, it is a hard axis limit — move the labware; clearing neighbors will not help. ([Reachability and Access](biomek-file-formats/deck-layouts/reachability-and-access.md)) |
| **A short `evalAmounts` does not zero the rest** | Wells past the end of the array take **element 0's** value, not `0.0`. Supply a full-length array with explicit zeros. An array *longer* than the well count is a hard error; a reservoir is section-length. ([Instrument Setup](step-documentation/03-Instrument-Setup.md)) |
| **`Cannot pipette …; the well only has 0.000 uL in it.`** | The enqueue-time volume ledger thinks the well is empty. Seed it with `volumeType` + `evalAmounts` if the liquid arrives off-deck — or account for earlier draws, which are cumulative. ([Concept Guide: Well Volume Tracking](concept-guides/05-well-volume-tracking.md)) |
| **`<where they came from>` is not a `DiscardTipsLocation` value** | It belongs to Fixed-8 Unload Tips `tipDestination` / Transfer `unloadLocation`. For tip-box routing use `DiscardTipsLocation: "<TipBox>"`. It fails silently on the wrong key. ([Instrument Setup](step-documentation/03-Instrument-Setup.md)) |
| **A Select Tips rearrange box must be declared *and* empty** | Declare a same-class tip box with `volumeType: "Known"` and a full-length `evalAmounts` of `0.0` — a tip box's inventory is its amounts array. Omitting the box is not the same as an empty box. ([Multichannel Select Tips](step-documentation/26-Multichannel-Select-Tips.md)) |
| **One MC Load Tips uses a whole box** | Multichannel skips partially-used boxes, so on a hybrid, a Span-8 draw from a shared box retires it from the MC pool. The Multichannel head can only reload the same tips multiple times if the tips are configured for reuse in the instrument setup step, or if using Select Tips steps. ([Multichannel Load Tips](step-documentation/22-Multichannel-Load-Tips.md)) |

### Position / Class Key by Family

| Family | Position field | Class field |
|--------|----------------|-------------|
| Fixed-8 | `where` | `what` |
| Span-8 | `where` | `what` |
| Multichannel | `location` | `labwareClass` |
| Transfer/Combine item | `position` | `labwareClass` |

> **Which positions exist on each deck** — the sample decks under
> [`biomek-file-formats/deck-layouts/samples/`](biomek-file-formats/deck-layouts/samples/)
> list every position name (and its ALP) on each instrument's default decks.
> However, an individual instrument may have a different deck, as defined in its
> instrument settings. Use them to look up valid `where` / `location` / `position` / `source` /
> `target` values before authoring.
