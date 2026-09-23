# Multichannel Load Tips

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Load Tips"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5 or i7 with a Multichannel pod |

---

## Behavior Summary

The Multichannel Load Tips step loads disposable tips onto all mandrels of the
Multichannel (96/384-channel) pod head. It locates an available tip box matching
the requested tips and loads tips from it.

Note that while tips must be loaded from a TipLoad1x1 ALP, the tip box does not
have to be present on that ALP. The step will move the tip box to an empty
TipLoad1x1 in order to load it.

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | `string` | — | Yes | *(must be provided)* | Name of the multichannel pod to use (e.g., `"Pod1"`). Must resolve to a multichannel pod. The MC pod is `"Pod1"` on a standalone Multichannel instrument (i5-MC, i7-MC) and on an i7 Hybrid; it is `"Pod2"` only when the MC is the right-hand pod of a dual-pod i7. Never author `"MC"`/`"MPod"`. See [Introduction to Biomek Method JSON](../Introduction-to-Biomek-Method-JSON.md#instrument-variants) §"Instrument Variants" / "Pod Types". |
| `tips` | `string` | — | Yes | *(must be provided)* | Tip labware class name, labware instance name, or deck position to load from (e.g., `"BC230"`, `"BC190F"`, `"TipBox6"`, `"P3"`). If a labware class or instance name is provided that matches more than one tip box, the step will select an appropriate tip box from the matching tip boxes. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

---

## Enumerated / Constrained Values

No enumerated constraints. The `tips` value must match a labware class name, a labware instance name, or a deck position name defined in the project's labware library / deck.

---

## Cross-Field Validation Rules

1. `pod` **must** resolve to a Multichannel pod.
2. `tips` **must** name a tip type retrievable on the deck (by class name, instance name, or deck position). **The deck-position form only works if that position currently holds a tip box.** Naming a position that exists on the deck but was left empty in Instrument Setup is not an error in itself — the value simply falls through to the class-and-instance search, which then fails reporting your position name as though it were a tip type (`Unable to find and retrieve tips of type TL1 for Pod1.`). If you get that message and the name in it is a deck position rather than a labware class, the position is empty; place a tip box there. If the value matches a stocked deck position, the step loads from that position directly; otherwise it searches all reachable tip boxes and, if none yields tips, the step fails and reports the reason it skipped each candidate box (used, partially used, named-instance mismatch when searched by class, unreachable, or no tip-load locations found).
3. The tip density **must** match the head: loading a 384-tip box onto a 96-channel head (or vice-versa) fails.
4. The tip box must be fully populated — all tips must be present. A
   partially-used box is skipped as partially used; an empty (fully consumed)
   box is skipped as used. Automatic tip-box retrieval will relocate a
   sealed/lidded box from storage and de-lid it (moving the lid to trash) before
   loading when no reachable open box is available.
5. The tip box must have remaining uses, as defined in the Instrument Setup
   step. See [Instrument Setup](03-Instrument-Setup.md), §`evalAmounts` for tip boxes.

> **Budgeting boxes: typically, one Load Tips consumes one box.** If the tip box
> is created with default settings in the Instrument Setup step, the tips may be
> loaded. This means that **every** Multichannel Load Tips step needs its own
> fully-populated box. The number of tip changes a Multichannel method can
> perform is therefore capped by how many boxes you can place, and how many
> `tL*` tip-load positions are available.
>
> To reuse tips from the same box, configure the number of usages in the
> Instrument Setup step ([Instrument Setup](03-Instrument-Setup.md), §`evalAmounts` for tip boxes).

> **On a hybrid instrument, keep each pod's boxes separate.** Where a Multichannel and a Span-8 pod
> share one deck, the two pods have opposite partial-box rules: Multichannel skips a partially-used
> box (rule 4), while Span-8 accepts one ([Span-8 Load Tips](17-Span-8-Load-Tips.md) rule 3).
> So a Span-8 load that draws a few tips from a shared box leaves that box partial and thereby
> **permanently removes it from the Multichannel pod's pool** — the box still looks full to a reader
> of the method, and no error is raised at the Span-8 step that consumed it. Declare dedicated boxes
> per pod, and point each pod's steps only at its own.
>
> The failure lands well away from its cause: it appears at the *later* Multichannel step as
> `Unable to find and retrieve tips of type <class> for <pod>`, reported against that step's source
> position rather than against the Span-8 load. Enqueue-time scheduling can also raise it before any
> tips have physically moved. If a hybrid method fails this way, audit which boxes the Span-8 steps
> touch before re-examining the Multichannel step.

---

## Structural Context

This step is a leaf and does not support subSteps.

---

## Canonical Examples

Load multichannel tips:

```json
{
  "stepType": "Multichannel Load Tips",
  "parameters": {
    "pod": "Pod1",
    "tips": "BC230"
  }
}
```

---

## Common Mistakes

- **Tip density mismatch with head**: A 384-tip box on a 96-channel head (or vice-versa) fails at load time.
- **Invalid tip name**: `tips` must match a labware class name, labware instance name, or deck position defined in the project — check Instrument Setup for available types. Silently misspelling produces the generic "unable to find and retrieve tips" error.
- **Assuming a partial/used box will do**: on a Multichannel pod, a partially-used box is always skipped and a fully-consumed box is reported as used; automatic retrieval will de-lid a fresh box only if one is reachable.
- **Searching a named box by class**: if a tip box on deck has been given a custom instance name, referencing it by its class alone is skipped as a named-instance mismatch — use the instance name instead.

---

## Related Guides

- **[Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md)** — tip class selection interacts with technique auto-selection on subsequent Multichannel Aspirate / Dispense / Mix steps.
