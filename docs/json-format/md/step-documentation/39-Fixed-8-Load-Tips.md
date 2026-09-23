# Fixed-8 Load Tips

| Property | Value |
|----------|-------|
| stepType | `"Fixed-8 Load Tips"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i3 (Fixed-8 pod required) |

## Behavior Summary

The Fixed-8 Load Tips step picks up disposable tips from a tip box on the deck
and attaches them to the Fixed-8 pod mandrels. It supports loading a specific
number of tips 1–8, automatically choosing which tips to load, or loading from a
specific mandrel position on a specific tip box.

When automatically choosing tips to load, and multiple tip boxes match the
specified tips, the step chooses the box with the fewest remaining tips, then
the furthest right, then the furthest back. Tip boxes must be 96-density (8 rows
× 12 columns).

Unless loading from specific positions is specified, tips are always loaded
beginning from mandrel 1. When loading less than 8 tips, the pod will offset to
the front of the tip box in order to position mandrel 1 inside the box. Be aware
that additional tip boxes may not be placed in front of the tip box being loaded
from in this scenario, or a collision error will be raised.

It is possible using the specific positions mode to offset the pod behind the
tip box, to load tips on e.g. mandrels 5-8 without tips on 1-4. In this case,
additional tip boxes may not be placed behind the tip box being loaded from, or
a collision error will be raised.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `tips` | `expression-capable string` | — | Yes | — | Identifies the tip source: deck position name (e.g. `"P1"`), tip box **instance** name (e.g. `"MyTips"`, may apply to multiple boxes), or tip box class name (e.g. `"BC230"`). |
| `mode` | `string` | — | **Yes** | `"NumberOfTips"` | Loading mode. See enumeration below. |
| `numberOfTips` | `expression-capable string` | — | Conditional | `"8"` | Number of tips to load (1–8). Used when `mode` is `"NumberOfTips"`. |
| `mandrel` | `integer` | — | Conditional | `1` | Mandrel number (1–8) that aligns with `mandrelPosition`. Used when `mode` is `"SpecificTips"`. The editor **never** authors a value other than `1` or `8`. Prefer using `1` or `8`. For example, a single tip can be loaded onto mandrel 1 by positioning mandrel `1` at `H1`, or onto mandrel 8 by positioning mandrel `8` at `A1`. |
| `mandrelPosition` | `expression-capable string` | — | Conditional | `"A1"` | Well position in tip box position `mandrel` over. Used when `mode` is `"SpecificTips"`. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

> _No `pod` key: Fixed-8 machines have a single Fixed-8 pod which the step resolves implicitly. Unlike Fixed-8 Aspirate/Dispense/Mix, do not add a `pod` key to this step._

## Enumerated / Constrained Values

### mode Values

| Value | Meaning |
|-------|---------|
| `"NumberOfTips"` | Load 1–8 tips (uses `numberOfTips` parameter) |
| `"SpecificTips"` | Load at explicit mandrel/position (uses `mandrel` and `mandrelPosition` parameters) |

## Cross-Field Validation Rules

1. `numberOfTips` **must** evaluate to an integer 1–8 — otherwise enqueue raises an invalid-tip-count error.
2. `SpecificTips` mode requires that `tips` resolves to exactly one position; if multiple positions match, enqueue raises an error.
3. The matched tip box must be 96-density (8 rows × 12 columns).
4. The tip box must have enough available tips for the requested count.

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

Load 8 tips (default mode):

```json
{
  "stepType": "Fixed-8 Load Tips",
  "parameters": {
    "tips": "P1",
    "mode": "NumberOfTips",
    "numberOfTips": "8"
  }
}
```

Load a single tip:

```json
{
  "stepType": "Fixed-8 Load Tips",
  "parameters": {
    "tips": "P1",
    "mode": "NumberOfTips",
    "numberOfTips": "1"
  }
}
```

Load at specific position in the tip box:

```json
{
  "stepType": "Fixed-8 Load Tips",
  "parameters": {
    "tips": "P1",
    "mode": "SpecificTips",
    "mandrel": 1,
    "mandrelPosition": "A3"
  }
}
```

## Common Mistakes

- **Tips already loaded on the pod**: this step requires empty mandrels; if any mandrel is currently tipped the step raises a tips-already-loaded error. Pair with an Unload Tips step first when re-loading mid-method.
- **`SpecificTips` mode with an ambiguous `tips` source** — the resolver requires exactly one matching position; a class name that matches multiple boxes raises an enqueue error.
- **Assuming any tip-box density works** — must be 96-density (8 × 12); other
  densities fail the geometry check.
