# Multichannel Unload Tips

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Unload Tips"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5 or i7 with a Multichannel pod |

---

## Behavior Summary

The Multichannel Unload Tips step removes disposable tips from the Multichannel (96/384-channel) pod head. If tips are currently loaded on the pod, the step unloads them; if no tips are loaded, the step does nothing.

---

## Where the tips go

This step takes **no unload-location parameter** — `pod` is its whole parameter set. The
destination is resolved by the pod from the deck and the tip box the loaded tips
came from. To change where tips are unloaded to, set the `DiscardTips` and
`DiscardTipsLocation` on the source tip box in the Instrument Setup step (see
[Instrument Setup](03-Instrument-Setup.md)).

- **No tips loaded** — the step does nothing.
- **Source tip box configured to discard to trash** (`DiscardTips` `true` with `DiscardTipsLocation` `<Any Trash>`, or no
  remembered source box) — the pod picks the **closest reachable position carrying the
  "Can Discard Tips" characteristic**. `Can Discard Tips` is a deck-position characteristic set in
  the instrument's deck configuration, not a step parameter. Reachability is **pod-specific**: on a
  two-pod instrument the other pod's bridge narrows each pod's usable X range. If no qualifying
  position is reachable by this pod, the step fails with:

  ```
  <pod> cannot find a location to discard tips.
  ```
  The pod name at the start identifies which pod's discard path failed. The search and the deck-side remedy are in [Reachability and Access](../biomek-file-formats/deck-layouts/reachability-and-access.md#symptom-to-likely-cause) §"Symptom to likely cause".
- **Source tip box configured to return tips to the tip box** (`DiscardTips` `false`, e.g. `DiscardTipsLocation: "<TipBox>"`) — the
  pod looks for the original box's current position, then for an **empty box of the same class**
  at the source position (not claimed by the other pod). If neither is found it falls back to the
  closest reachable trash. No reachable trash at that point raises a "cannot find the tip box"
  error. A trashless deck is still a risk on the return-to-box path.
- **Tips loaded via a Select-tips step** — if the selected tips cannot be placed back into the source box, the pod
  falls back to the closest reachable trash on the same terms. Note that tips
  loaded via Select Tips steps do NOT decrement the tip usage count. It is
  recommended not to share tip boxes between Select Tips steps and regular
  Multichannel tip load/unload steps.

A named destination must be a position the pod can legitimately unload at: if it holds no labware
it must carry the "Can Discard Tips" characteristic, otherwise the unload is rejected.

The `DiscardTips` / `DiscardTipsLocation` routing keys live on the **tip-box labware object** in
[Instrument Setup](03-Instrument-Setup.md), not on this step.

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | `string` | — | Yes | *(must be provided)* | Name of the multichannel pod (e.g., `"Pod1"`). Must resolve to a Multichannel pod. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

---

## Enumerated / Constrained Values

No enumerated constraints.

---

## Cross-Field Validation Rules

1. `pod` **must** resolve to a Multichannel pod.
2. A discard location **must** be reachable **by this pod** (pod-specific, narrowed by the other pod's bridge on two-pod instruments). See [Where the tips go](#where-the-tips-go) and [Span-8 Unload Tips](18-Span-8-Unload-Tips.md#deck-prerequisite) "Deck prerequisite". If no motion path exists, the failure is wrapped as an unload-tips path error; a missing position yields a "deck position not found" error.

---

## Structural Context

This step is a leaf and does not support subSteps.

---

## Canonical Examples

Unload multichannel tips:

```json
{
  "stepType": "Multichannel Unload Tips",
  "parameters": {
    "pod": "Pod1"
  }
}
```

---

## Common Mistakes

- **No reachable discard location for *this* pod**: ensure a `Can Discard Tips` position is reachable by the pod named on this step, not merely present on the deck. No step parameter can substitute; see [Where the tips go](#where-the-tips-go).
- **Assuming `DiscardTipsLocation: "<TipBox>"` removes the need for a trash**: the return-to-box path still falls back to the closest reachable trash whenever the original box is unavailable, so it can fail on a trashless deck with a "cannot find the tip box" error instead.

---

## Related Guides

- **[Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md)** — tip lifecycle context; pairs with [Multichannel Load Tips](22-Multichannel-Load-Tips.md).
