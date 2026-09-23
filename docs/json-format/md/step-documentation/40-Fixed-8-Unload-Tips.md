# Fixed-8 Unload Tips

| Property | Value |
|----------|-------|
| stepType | `"Fixed-8 Unload Tips"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i3 (Fixed-8 pod required) |

## Behavior Summary

The Fixed-8 Unload Tips step removes tips from the Fixed-8 pod mandrels. It supports returning tips to their original box, discarding to trash, or placing at a specific location. If no tips are currently loaded, the step is a no-op (succeeds silently). The step supports two modes: AutomaticSelection (finds an available position) and SpecificLocations (unloads to an explicit mandrel/position in the tip box).

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `tipDestination` | `expression-capable string` | — | No | `"<where they came from>"` | Where to put the tips. Accepts a deck position name, tip box **instance** name, tip box class name, or a special token (see below). |
| `mode` | `string` | — | No | `"AutomaticSelection"` | Unload mode. See enumeration below. |
| `mandrel` | `integer` | — | Conditional | `1` | Mandrel number (1–8) that aligns with `mandrelPosition`. Used when `mode` is `"SpecificLocations"`. The editor **never** authors a value other than `1` or `8`. Prefer using `1` or `8`. |
| `mandrelPosition` | `expression-capable string` | — | Conditional | `"A1"` | Target position in tip box. Used in SpecificLocations mode. |
| `isTipsTrash` | `boolean` | — | No | `false` | **Inert UI state — not read at runtime.** Preserve for round-trip fidelity, but do not rely on it to route unloads to trash — use `tipDestination` (`"<Any Trash>"` or a trash-position name) instead. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

> _No `pod` key: Fixed-8 machines have a single Fixed-8 pod which the step resolves implicitly. Unlike Fixed-8 Aspirate/Dispense/Mix, do not add a `pod` key to this step._

## Enumerated / Constrained Values

### tipDestination Special Tokens

| Value | Meaning |
|-------|---------|
| `"<where they came from>"` | Return tips to the position they were loaded from |
| `"<Any Trash>"` | Discard tips to any available trash position. Requires a deck that defines one; not every deck does (see rule 5). |
| `"<Use Original Box Settings>"` | Check the source tip box's discard settings configured in the Instrument Setup step |

Any other string value is resolved against deck position names, tip box instance
names, and tip box class names (in that precedence order).

### mode Values

| Value | Meaning |
|-------|---------|
| `"AutomaticSelection"` | Auto-find suitable location in the tip box |
| `"SpecificLocations"` | Unload to explicit mandrel/position. Cannot be used with trash positions. |

## Cross-Field Validation Rules

1. If no tips are loaded on the pod, the step succeeds immediately (no-op).
2. `SpecificLocations` mode requires that `tipDestination` resolves to exactly one position; if multiple match, enqueue raises an error.
3. The destination must have empty spaces corresponding to the loaded tips, or be a trash position.
4. When `tipDestination` is `"<Use Original Box Settings>"`, the step reads the
   source box's discard-tips settings (configured in Instrument Setup) to determine the actual destination.
5. When `tipDestination` is `"<Any Trash>"` (or resolves to trash via `"<Use
   Original Box Settings>"`), the step searches for any deck position carrying
   the "Can Discard Tips" characteristic.
6. When `mode` is `SpecificLocations`, `tips` cannot point to a trash position.

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

Return tips to their source box:

```json
{
  "stepType": "Fixed-8 Unload Tips",
  "parameters": {
    "tipDestination": "<where they came from>"
  }
}
```

Discard tips to trash:

```json
{
  "stepType": "Fixed-8 Unload Tips",
  "parameters": {
    "tipDestination": "<Any Trash>"
  }
}
```

Unload to a specific position in the box:

```json
{
  "stepType": "Fixed-8 Unload Tips",
  "parameters": {
    "tipDestination": "P1",
    "mode": "SpecificLocations",
    "mandrel": 1,
    "mandrelPosition": "A5"
  }
}
```

## Common Mistakes

- **Relying on `isTipsTrash` to enable trash behavior**: `isTipsTrash` is
  **inactive — not read at runtime**. To discard tips, set `tipDestination` to
  `"<Any Trash>"` or a trash position (e.g. `"TR1"`), and use `mode` `"AutomaticSelection"`.
- **Trying to "empty" a tip box to make it a trash**: a position is a
  trash/discard target only when it has the "Can Discard Tips" deck
  characteristic (e.g. `"<Any Trash>"` / `TR1`). A tip box placed through
  `deckItems` in Instrument Setup tracks empty slots and accepts returned tips
  at specific row/column, but is not a trash. Routing unloads to a tip box
  returns tips there; it does not empty it.
- **Silent no-op when no tips are loaded**: the step succeeds without error even when there is nothing to unload — useful for defensive cleanup, but hides ordering bugs where tips were expected on the pod.
- **Misspelling special tokens**: the tokens are case-insensitive but must match the exact bracketed text (e.g., `"<where they came from>"`).
