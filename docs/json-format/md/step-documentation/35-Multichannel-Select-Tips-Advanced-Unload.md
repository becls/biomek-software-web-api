# Multichannel Select Tips Advanced Unload

> **Reference manual:** Chapter 20 documents these as *Select Tips* steps (the manual omits the *Multichannel* prefix); the JSON `stepType` keeps the `Multichannel Select Tips …` name.

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Select Tips Advanced Unload"` |
| Category | Leaf (does not accept child steps) |
| Terminator | N/A |
| Compatible Hardware | Multichannel pod only |

## Behavior Summary

The Advanced Unload Tips step unloads tips to a specific row and column position
in a tip box, giving explicit control over where tips are placed. The step verifies
that tips were originally loaded via a Select Tips family load step
(`Multichannel Select Tips Load` or `Multichannel Select Tips Advanced Load` —
not via standard multichannel tip loading), then places them at the specified
location.

This step must be placed inside a `"Multichannel Select Tips"` container.

Note that Multichannel Select Tips steps do not decrement the tip reuse counter,
so after unloading tips, the tips are available to reuse.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `tipsLocation` | `expression-capable string` | — | Yes | `""` | Deck position containing the tip box, tip box type, or tip box **instance** name. |
| `row` | `expression-capable string` | — | Yes | `"1"` | Row number in tip box to position mandrel A1 over (1-based; negative or zero values are allowed). |
| `column` | `expression-capable string` | — | Yes | `"1"` | Column number in tip box to position mandrel A1 over (1-based; negative or zero values are allowed). |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

## Cross-Field Validation Rules

1. Pod **must** have tips loaded, or an unload-no-tips error is raised.
2. Tips **must** have been loaded via a Select Tips Load or Advanced Load step.
3. Destination must contain a tip box with matching tip type.
4. `row` must evaluate to a valid integer.
5. `column` must evaluate to a valid integer.
6. The destination must have space in the box for the tips to unload.
7. The position of mandrel A1 must not leave any loaded tips outside of the box.

## Structural Context

This step is a leaf inside a `"Multichannel Select Tips"` container.

## Canonical Examples

```json
{
  "stepType": "Multichannel Select Tips Advanced Unload",
  "parameters": {
    "tipsLocation": "TL1",
    "row": "1",
    "column": "5"
  }
}
```

## Common Mistakes

- **Tips not loaded via a Select Tips Load or Advanced Load step**: Only tips whose source is tracked can be advanced-unloaded; tips picked up by a plain multichannel Load Tips will be rejected.
- **Destination tip box has the wrong type**: The box at `tipsLocation` must
  accept the currently-loaded tip type; there is no auto-conversion.
- **Destination tip box must have empty cells to unload all of the tips to**:
  The loaded tips must be entirely over the target box, and each cell a tip is
  over must be empty.
