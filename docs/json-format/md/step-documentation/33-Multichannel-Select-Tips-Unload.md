# Multichannel Select Tips Unload

> **Reference manual:** Chapter 20 documents these as *Select Tips* steps (the manual omits the *Multichannel* prefix); the JSON `stepType` keeps the `Multichannel Select Tips …` name.

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Select Tips Unload"` |
| Category | Leaf (does not accept child steps) |
| Terminator | N/A |
| Compatible Hardware | Multichannel pod only |

## Behavior Summary

The Select Tips Unload step removes tips from the multichannel pod to a given location. If the destination is a tip trash, tips are simply discarded. If the destination is a tip box, the step finds an available row/column position (first-fit scan) and returns the tips to the box. The step determines the unload position automatically based on the current tip pattern.

This step must be placed inside a `"Multichannel Select Tips"` container.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `unloadPosition` | `expression-capable string` | — | Yes | `""` | Location to unload tips to (tip trash, position name, tip box class, or tip box instance name). |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

## Cross-Field Validation Rules

1. Pod **must** have tips loaded, or an unload-no-tips error is raised.
2. `unloadPosition` must evaluate to a tip trash, position containing a tip box,
   or a tip box instance, or an invalid-unload-position error is raised.
3. If unloading to a tip box, it must have space for the current tip pattern, or a no-space error is raised.
4. When unloading to a tip box, the tips must have been loaded via a `"Multichannel Select Tips Load"` step, or a wrong-tip-load error is raised.

## Structural Context

This step is a leaf inside a `"Multichannel Select Tips"` container.

## Canonical Examples

Unload to tip trash:

```json
{
  "stepType": "Multichannel Select Tips Unload",
  "parameters": {
    "unloadPosition": "TR1"
  }
}
```

Return tips to named box:

```json
{
  "stepType": "Multichannel Select Tips Unload",
  "parameters": {
    "unloadPosition": "MyTips"
  }
}
```

## Common Mistakes

- **Tip box lacks space for the current pattern**: The step auto-picks the destination cells, but the box must actually have those cells free — otherwise enqueue throws.
- **Trying to unload to a plate/reservoir**: `unloadPosition` must resolve to a tip trash or a tip box; any other labware fails.

## Related Steps

- **[Multichannel Select Tips](26-Multichannel-Select-Tips.md)** — the parent container; explains the tips-off bracket and valid child steps.
