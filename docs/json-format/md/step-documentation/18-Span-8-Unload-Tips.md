# Span-8 Unload Tips

| Property | Value |
|----------|-------|
| stepType | `"Span-8 Unload Tips"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7 (Span-8 pod required) |

## Behavior Summary

The Span-8 Unload Tips step discards disposable tips from the Span-8 pod's selected probes. Fixed-tip probes in the selection are silently skipped. Typically executed at the end of a pipetting sequence or when switching tip types. Unload tips before the method's Finish step.

## Where the tips go

This step takes **no unload-location parameter**. The destination is resolved by the pod from the
deck and the tip box the loaded tips came from. Resolution order:

1. If the pod does not remember a source tip box, the location is `<Any Trash>`.
2. Otherwise the pod reads the `DiscardTips` and `DiscardTipsLocation` routing keys off the
   **tip-box labware object** configured in the [Instrument Setup](03-Instrument-Setup.md) step —
   not off this step. Span-8 defaults `DiscardTips` to `true` and `DiscardTipsLocation` to
   `<Any Trash>` when the box does not set them.
3. A **named** position is used only when `DiscardTips` is `true` **and** `DiscardTipsLocation` is
   a position name. That position must exist on the deck, or the step raises a "deck position does
   not exist" error:`<pod> cannot discard tips. Deck position "<position>" does not exist.`
   The pod moves directly to the position, using the position's Span-8 tip-disposal offsets, so it must be a real
   tip-disposal position.
4. In every other case the pod searches the deck for the **closest position it can reach that
   carries the "Can Discard Tips" characteristic** — a tip-trash / waste ALP. If no such position
   is both present and reachable by this pod, the step raises the "tip discard location" error:

   ```
   <pod> is unable to find a tip discard location.
   ```
   This differs from the named-position error above; the wording distinguishes a missing name from a missing/unreachable trash.

**The Span-8 pod has no return-to-box path for disposable tips.** `DiscardTips: false` and
`DiscardTipsLocation: "<TipBox>"` (which sets `DiscardTips` to `false`) still force the trash
search, producing the same "tip discard location" error on a deck with no reachable trash.

### Deck prerequisite

A method that loads and unloads Span-8 disposable tips requires a tip-trash / waste ALP with the
"Can Discard Tips" characteristic within the Span-8 pod's reach. **"Can Discard Tips" is a
deck-position characteristic, not a step parameter.** It is set in the instrument's deck
configuration (Deck Editor or factory scripts); this step can only reference positions that already
exist. On a deck with no reachable tip trash, no step parameters or tip-box routing keys can resolve the problem — the remedy is in Instrument Setup or the deck layout.

**Reachability is pod-specific.** The auto-search filters candidates through this pod's own reach
check, and on a two-pod instrument the other pod's bridge narrows that reach — so a trash present on
the deck can still be unreachable here. Move it toward this pod's side or reassign which pod
unloads; see [Reachability and
Access](../biomek-file-formats/deck-layouts/reachability-and-access.md#two-pods-on-one-bridge) §"Two pods on one
bridge".

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | `string` | — | Yes | `""` | Name of the Span-8 pod (e.g., `"Pod1"`). Must resolve to a Span-8 pod. |
| `useProbes` | comArray (boolean[8]) | — | No | *(all true)* | 8-element boolean array (indices 0–7), one per Span-8 mandrel. `true` = probe participates. Serialized as `{"_biomekType": "comArray", "arraySubtype": "boolean", "values": [...]}`. |
| `mandrelExpression` | `string` | — | No | `""` | Expression-based probe selection. Used when `useExpression` is `true`. |
| `useExpression` | `boolean` | — | No | `false` | If `true`, use `mandrelExpression` instead of `useProbes` for probe selection. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

## Enumerated / Constrained Values

No enumerated constraints.

## Cross-Field Validation Rules

1. `pod` **must** name a pod that exists on the instrument and resolve to a Span-8 pod. An unresolvable name raises an "unknown pod" error; a wrong pod type raises a validation error.
2. A tip discard location **must** be reachable **by this pod** (pod-specific, narrowed by the other pod's bridge on two-pod instruments). See [Where the tips go](#where-the-tips-go) and [Deck prerequisite](#deck-prerequisite).
3. When `useExpression` is `true`, `mandrelExpression` is used for probe selection; otherwise `useProbes` is used.

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

> The examples use `"pod": "Pod1"`. The Span-8 pod is `Pod1` on a standalone Span-8 instrument
> (i5-Span, i7-Span) and `Pod2` (the second/rightmost pod) on an i7 Hybrid. **Do not treat
> `"Pod2"` as canonical** — read the pod name from the target instrument's configuration.

Unload tips from all probes:

```json
{
  "stepType": "Span-8 Unload Tips",
  "parameters": {
    "pod": "Pod1"
  }
}
```

Unload tips from specific probes:

```json
{
  "stepType": "Span-8 Unload Tips",
  "parameters": {
    "pod": "Pod1",
    "useProbes": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true, true, true, true, false, false, false, false]
    },
    "useExpression": false
  }
}
```

## Common Mistakes

- **Naming the wrong pod for the instrument family** — `Pod1` on standalone Span-8, `Pod2` on i7 Hybrid; a Multichannel pod name resolves but fails the Span-8 pod check at Enqueue.
- **Setting `useExpression: true` but leaving `mandrelExpression` empty** — the `useProbes` array is then ignored and the expression evaluates to an empty probe set (silent no-op, no error). Author both keys as a pair.
- **No reachable tip-discard location for *this* pod** — raises the "tip discard location" error. No step parameter can fix this; see [Deck prerequisite](#deck-prerequisite).
- **Trying to steer the discard from this step** — this step has no location key. The only routing controls are `DiscardTips` / `DiscardTipsLocation` on the tip-box labware in [Instrument Setup](03-Instrument-Setup.md), and on Span-8 they select *which* trash, never a return to the box.
- **Expecting `DiscardTipsLocation: "<TipBox>"` to avoid the trash on Span-8** — it forces the trash search instead. `<TipBox>` returns tips to their box on the Multichannel pod only.
- **Assuming this step throws when probes carry no disposable tips** — the step silently skips probes that have no tips or that carry fixed tips; when no probe qualifies, no pod motion occurs and no error is raised.
