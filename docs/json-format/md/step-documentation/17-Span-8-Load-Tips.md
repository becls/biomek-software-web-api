# Span-8 Load Tips

| Property | Value |
|----------|-------|
| stepType | `"Span-8 Load Tips"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7 (Span-8 pod required) |

## Behavior Summary

The Span-8 Load Tips step loads disposable tips onto the Span-8 pod's probes. It first discards any currently loaded tips on the selected probes, then finds and loads new tips of the specified type from available tip boxes on the deck. The probe selection determines which of the 8 probes participate — either via a boolean array or a mandrel expression.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | `string` | — | Yes | `""` | Name of the Span-8 pod (e.g., `"Pod1"`). Must resolve to a Span-8 pod. |
| `tips` | `string` | — | Yes | `""` | Tip labware class name, labware instance name, or deck position to load from (e.g., `"BC80"`, `"BC230_LLS"`, `"TipBox6"`, `"P3"`). The step first checks whether the value names a deck position (loading from it directly); otherwise it searches all reachable tip boxes whose class or instance name matches. Supports expressions (prefix with `=`). |
| `useProbes` | comArray (boolean[8]) | — | No | *(all true)* | 8-element boolean array (indices 0–7), one per Span-8 mandrel. `true` = probe participates. Serialized as `{"_biomekType": "comArray", "arraySubtype": "boolean", "values": [...]}`. |
| `mandrelExpression` | `string` | — | No | `""` | Expression-based probe selection using **1-indexed** probe numbers (e.g., `"1,3,4"` selects probes 1, 3, and 4). Also accepts Biomek expressions (e.g., `"=myProbeVar"`). Used when `useExpression` is `true`. Probe numbers below 1 raise a "must be greater than zero" error. |
| `useExpression` | `boolean` | — | No | `false` | If `true`, use `mandrelExpression` instead of `useProbes` for probe selection. |

> _The probe-selection keys (`useProbes`, `mandrelExpression`, `useExpression`) stay Optional — omission does not throw; the selector falls back to the standard all-probes-selected default._

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

## Enumerated / Constrained Values

No enumerated constraints beyond those listed in the table.

## Cross-Field Validation Rules

1. `pod` **must** resolve to a Span-8 pod.
2. When `useExpression` is `true`, `mandrelExpression` is used for probe selection; otherwise `useProbes` is used. The selected probe subset is honored — only those probes are loaded.
3. `tips` **must** name a tip type retrievable on the deck (by class name, instance name, or deck position). **The deck-position form only works if that position currently holds a tip box.** A position that exists but was left empty in Instrument Setup falls through to the class-and-instance search, which then fails reporting your position name as though it were a tip type (`Unable to find and retrieve tips of type TL4 for Pod2.`). When the name in that message is a deck position rather than a labware class, the position is empty; place a tip box there. If the value matches a stocked deck position, the step loads from that position directly; otherwise it searches reachable tip boxes and, if none yields tips, raises a "unable to find tips" error with per-box skip reasons appended. Partially used boxes are accepted (unlike the Multichannel path). Automatic tip-box retrieval will relocate and de-lid a sealed box from storage when no reachable open box is available.
4. Loading onto probes that carry non-disposable (fixed) tips raises a "cannot load on fixed tips" error — see the probe-selection note above the second example. Whether a probe is fixed or disposable is **instrument state, not a method choice**: the pod reads it from its own per-probe tip configuration, and no key in this step (or any other) can override it. An instrument whose Span-8 probes are all configured with fixed tips will reject this step on every probe selection. Check the target instrument's `podSettings.<pod>.settings.tip1…tip8` before authoring — see [Fixed vs. disposable probes](../biomek-file-formats/instrument-settings/pod-settings-format-spec.md#fixed-vs-disposable-probes).

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

> The examples use `"pod": "Pod1"`. The Span-8 pod is `Pod1` on a standalone Span-8 instrument
> (i5-Span, i7-Span) and `Pod2` (the second/rightmost pod) on an i7 Hybrid. **Do not treat
> `"Pod2"` as canonical** — read the pod name from the target instrument's configuration.

Load tips on all probes:

```json
{
  "stepType": "Span-8 Load Tips",
  "parameters": {
    "pod": "Pod1",
    "tips": "BC230_LLS",
    "useProbes": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true, true, true, true, true, true, true, true]
    },
    "useExpression": false
  }
}
```

`useProbes` is defaulted to "all true" internally but the enqueue path for Span-8 pods that
mix fixed and disposable probes routes based on this array; omitting it can raise a "cannot load on fixed tips" error when the default routing chooses a fixed-tip probe. Include
the explicit boolean array for a disposable-tip method.

Load tips on specific probes:

```json
{
  "stepType": "Span-8 Load Tips",
  "parameters": {
    "pod": "Pod1",
    "tips": "BC80",
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

- **Omitting `useProbes` on a pod that mixes fixed and disposable probes** — the "all-true" internal default can route through a fixed probe and raise a "cannot load on fixed tips" error. Always author the explicit array on such pods.
- **Authoring a disposable-tip method against a fixed-tip instrument** — narrowing `useProbes`, changing `tips`, or adding an unload first will not help: every probe is fixed, and the method has no way to ask for disposable ones. The instrument's probe configuration must be changed (Hardware Setup), or the method must run on an instrument already configured for disposable tips. See [Fixed vs. disposable probes](../biomek-file-formats/instrument-settings/pod-settings-format-spec.md#fixed-vs-disposable-probes).
- **Setting `useExpression: true` but leaving `mandrelExpression` empty** — the `useProbes` array is then ignored and there is no probe selection to consume; author both keys as a pair.
- **Naming the wrong pod for the instrument family** — `Pod1` on standalone Span-8 (i5-Span/i7-Span), `Pod2` on i7 Hybrid. A Multichannel pod name resolves but fails the Span-8 pod check.
- **Assuming a used tip box will be re-picked** — boxes whose tips have all been consumed are skipped with a "has been used" skip reason. The step searches for unused or partially used boxes on the deck, and can automatically retrieve and de-lid a sealed box from storage if no open box is reachable.
- **Searching a named box by its class** — if Instrument Setup gave the tip box a name, `tips` must be that name; its class alone is skipped as a named-instance mismatch, and the error names the box's own name back to you. Author the instance name (or the box's deck position), not the class.
- **Naming a deck position that has no box** — the position-name form of `tips` only works when that position actually holds a tip box. A real-but-empty position falls through to the class search and fails with your position name reported as though it were a tip type.
