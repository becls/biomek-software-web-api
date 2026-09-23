# Move Labware

| Property | Value |
|----------|-------|
| stepType | `"Move Labware"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | Any instrument with at least one pod that has a gripper (rotating offset gripper or Fixed-8 integrated gripper). |

## Behavior Summary

The Move Labware step moves labware from one deck position to another using a pod-mounted gripper. At enqueue time the step resolves the named pod (`pod`) and verifies that it has a gripper; if it does not, the step raises an error. It then resolves the `source` and `target` deck position names against the instrument's deck. The step supports three stacking modes: move the entire stack (default when `depth` is unbound and `leaveBottomLabware` is false), move all but the bottom piece (`leaveBottomLabware` = true), or move a specific number of pieces from the top (`depth` bound to a numeric value). A `gripSide` parameter controls whether the gripper fingers point toward `"A1 near"` (right) or `"A1 away from"` (left); for Fixed-8 pods the grip-side value is accepted but silently ignored, defaulting to A1-near. The step supports gripper offset overrides (`xOffset`, `yOffset`, `zOffset`, `squeeze`, `unsqueeze`) at three precedence levels: step-parameter keys override position/labware-class gripper-info overrides, which override instrument defaults (unset). Position-level overrides are resolved via `GripperInfoOverride` keys on the source and destination positions paired with a `GripperInfoOverrides` dictionary on the labware class.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | string | — | Yes | `""` | Name of the pod to use for the move (e.g., `"Pod1"`). The pod must have an attached gripper. |
| `source` | string | — | Yes | `""` | Name of the deck position that contains the labware to move (e.g., `"P13"`). Supports expressions. Valid position names come from the selected deck — see the sample decks under [`deck-layouts/samples/`](../biomek-file-formats/deck-layouts/samples/). |
| `target` | string | — | Yes | `""` | Name of the deck position to move the labware to (e.g., `"P23"`). The target position should be empty or capable of receiving additional labware; the gripper will raise an error if the move cannot be completed. Supports expressions. |
| `gripSide` | string | — | Yes | *(none)* | Side of the gripper to use: `"A1 near"` or `"A1 away from"`. **Key must be present for every pod**, including Fixed-8. An absent key raises a "could not find GripSide" system error. For Fixed-8 pods, an empty or invalid *value* is silently coerced to A1-near; for offset grippers an empty value raises a "missing grip side" error. See the Enumerated / Constrained Values section. |
| `leaveBottomLabware` | boolean | — | Conditional (when `depth` is unbound) | *(none — required when depth is unbound)* | When `true` and `depth` is unbound, moves all labware except the bottom piece. Source must contain more than one labware item. Omission raises a "could not find LeaveBottomLabware" system error. If `depth` **is** bound, `leaveBottomLabware` is never read and may be omitted freely. |
| `depth` | numeric | count | No | *(unbound)* | Number of labware pieces to move from the top of the stack. When bound, overrides `leaveBottomLabware`. Not bound by default — leaving it unbound moves the entire stack (subject to `leaveBottomLabware`). |
| `xOffset` | numeric | cm | No | *(unbound)* | Gripper X-axis offset override. When bound, takes precedence over position/labware-class gripper-info overrides. |
| `yOffset` | numeric | cm | No | *(unbound)* | Gripper Y-axis offset override. |
| `zOffset` | numeric | cm | No | *(unbound)* | Gripper Z-axis offset override. Used for moves to/from special positions like a Static Peltier ALP. |
| `squeeze` | numeric | cm | No | *(unbound)* | Gripper squeeze (inward clamp distance) override. |
| `unsqueeze` | numeric | cm | No | *(unbound)* | Gripper unsqueeze (outward release distance) override. |

> **Note on offset keys**: The five offset keys (`xOffset`–`unsqueeze`) are programmatic-only; they do not appear in the GUI and are set only when creating steps via scripting or JSON authoring. Values are in centimeters and override (replace) any position/labware-class gripper-info overrides — bind only the axes to displace, and omit (rather than set to `0`) the ones to inherit.

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

## Enumerated / Constrained Values

### `gripSide`

| Value | Meaning |
|-------|---------|
| `"A1 near"` | Gripper fingers point to the right (toward A1 side). |
| `"A1 away from"` | Gripper fingers point to the left (away from A1 side). |

For offset (rotating) grippers, `gripSide` must be set to one of the two values above; an empty or invalid value raises an error. For Fixed-8 pods (pod type `"Fixed8"`), any value including empty or invalid is silently coerced to A1-near — the parameter is accepted for compatibility but has no physical effect.

> **Step UI dynamically filters grip sides by physical reachability**: the editor's "Holding labware with" dropdown lists only the grip side(s) that are physically reachable for the chosen source/destination. When both sides are reachable it lists both; when only one is, it lists only that one; when neither is reachable the dropdown is empty and the UI displays an error indicating the move cannot be carried out. This is editor validation — at enqueue an out-of-reach move fails with a "pipettor destination X/Y … is outside of travel range" error (see Cross-Field Validation Rule 11).

### Stacking mode (derived from `depth` and `leaveBottomLabware`)

| `depth` bound? | `leaveBottomLabware` | Behavior |
|----------------|----------------------|----------|
| No | `false` | Moves the entire stack. |
| No | `true` | Moves all but the bottom piece. Requires stack depth > 1. |
| Yes | *(ignored)* | Moves exactly `depth` pieces from the top of the stack. |

## Cross-Field Validation Rules

1. **Pod must have a gripper**: The named `pod` must be installed on the instrument and must have a gripper. If the pod has no gripper or the gripper object is null, the step raises an "invalid pod type" error.
2. **Source must be specified and found**: `source` must be a non-empty string that resolves to a position on the deck. An empty string or unresolved name raises a source-not-specified or source-not-found error.
3. **Target must be specified and found**: Same rules apply to `target`.
4. **Source must not be empty**: The source position must contain at least one labware item. Otherwise the step raises a source-is-empty error.
5. **Grip side required for offset grippers**: When the pod is not Fixed-8, `gripSide` must be `"A1 near"` or `"A1 away from"`. An empty value raises a "gripper cannot perform this move" error; any other invalid value raises an "unable to interpret grip side" error.
6. **Leave-bottom requires depth > 1**: When `leaveBottomLabware` is `true` and `depth` is unbound, the computed depth is the stack height minus one. If the stack has only one piece, this yields zero and the step raises a "cannot leave bottom labware" error.
7. **Depth must not exceed actual stack depth**: When `depth` is bound, it is used directly. If `depth` exceeds the actual stack depth, the step fails with a source-depth-vs-stack-height mismatch.
8. **Grip-info overrides must agree**: If both the source and destination positions have a `GripperInfoOverride` key, the values must be identical (case-insensitive). Mismatched values raise an incompatible-grip-override error.
9. **Labware class must contain named override**: If a `GripperInfoOverride` value is resolved, the labware class being gripped must have a matching key under its `GripperInfoOverrides` dictionary. If the key is not found, the step raises a bad-labware-class-grip-override error.
10. **Step-parameter offsets take final precedence**: If any of `xOffset`, `yOffset`, `zOffset`, `squeeze`, or `unsqueeze` is bound in the step's parameters, it overrides the corresponding value from position/labware-class gripper-info overrides.
11. **Resolved position must be within the pod's travel range**: The final gripper coordinate (deck position plus any offsets) must lie inside the pod's physical X/Y travel limits. A position outside the limit fails at enqueue with a "destination outside travel range" error. Discarding to a far-corner trash or applying large offsets are the common triggers.
12. **Labware class must carry `Moveable`**: The class of the labware being moved must carry the `Moveable` characteristic. A class that lacks it fails at enqueue with `Labware of type <class> cannot be moved.` See [Catalog: Labware Classes](../biomek-file-formats/project-items/catalog-labware-classes.md) for which default classes lack it (notably `BCDeep96Square` and the wash stations among titer plates; most tube racks; four of the seven reservoirs).

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

### Example 1: Simple move of a single plate

```json
{
  "stepType": "Move Labware",
  "parameters": {
    "pod": "Pod1",
    "source": "P13",
    "target": "P23",
    "gripSide": "A1 near",
    "leaveBottomLabware": false
  }
}
```

### Example 2: Move top 2 plates from a stack

```json
{
  "stepType": "Move Labware",
  "parameters": {
    "pod": "Pod1",
    "source": "P5",
    "target": "P10",
    "gripSide": "A1 away from",
    "leaveBottomLabware": false,
    "depth": 2
  }
}
```

### Example 3: Move stack leaving bottom labware in place

```json
{
  "stepType": "Move Labware",
  "parameters": {
    "pod": "Pod1",
    "source": "P3",
    "target": "P8",
    "gripSide": "A1 near",
    "leaveBottomLabware": true
  }
}
```

### Example 4: Move with gripper offset overrides (e.g., Static Peltier destination)

```json
{
  "stepType": "Move Labware",
  "parameters": {
    "pod": "Pod1",
    "source": "P7",
    "target": "P15",
    "gripSide": "A1 near",
    "leaveBottomLabware": false,
    "zOffset": 0.5,
    "squeeze": 0.1,
    "unsqueeze": 0.1
  }
}
```

## Common Mistakes

- **`gripSide` required even for Fixed-8**: The key is read at the top of enqueue before the pod-family branch, so omission throws a "could not find GripSide" system error on any pod. Only the *value* is ignored on Fixed-8.
- **Binding `depth` alongside `leaveBottomLabware: true`**: A bound `depth` silently wins — `leaveBottomLabware` is not consulted. Use one mode, not both.
- **Binding an offset to `0` instead of omitting it**: `0` is a bound override that displaces the position/labware-class default; omit the key to inherit.
- **Mismatched `GripperInfoOverride` on source vs. destination**: Both positions defining the same key with different values raises an incompatible-grip-override error. Match them, or define only one side.
- **Large offsets or corner discards past travel range**: Deck-corner trash or aggressive offsets can push the resolved coordinate outside the pod's X/Y limits, failing at enqueue with a "destination outside travel range" error.
- **Referencing a moved plate by its old position**: after this step relocates a plate, a downstream step that names the plate's *original* deck position finds it empty and fails. References by the plate's **instance name** (its `properties.name` from Instrument Setup) or class name auto-resolve to its current position and need no update — prefer naming labware over hard-coding positions across a Move.
