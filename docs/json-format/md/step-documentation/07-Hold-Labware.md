# Hold Labware

| Property | Value |
|----------|-------|
| stepType | `"Hold Labware"` |
| Category | Free-form container |
| Terminator | `"End"` (caption `"End Holding Labware"`, helpContextID 141) |
| Compatible Hardware | i5, i7 |

---

## Behavior Summary

The Hold Labware step picks up labware from a source deck position using a gripper attached to a named pod, holds the labware in the gripper while all child steps execute, then places the labware back at the original source position. The step supports three stacking modes: pick up a specific number of pieces from the top of the stack (`depth` bound), pick up all but the bottom piece (`moveEntireStack` false and `depth` unbound), or pick up the entire stack (`moveEntireStack` true). Gripper offset overrides (`gripperXOffset`, `gripperYOffset`, `gripperZOffset`, `squeeze`, `unsqueeze`) may be specified at the step level to override position- and labware-class-level gripper information. Typical use cases include de-lidding a plate to pipette to it and then replacing the lid, or lifting labware from a stack to access the bottom item.

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | string | — | Yes | `""` | Name of the pod whose gripper will hold the labware (e.g., `"Pod1"`). The pod must have an attached gripper. Supports expressions. |
| `source` | string | — | Yes | `""` | Deck position name from which the labware is picked up and to which it is returned (e.g., `"P5"`, `"TL3"`). Must contain labware at runtime. Supports expressions. |
| `gripSide` | string | — | Yes | *(unbound)* | Side of the gripper to use: `"A1 near"` or `"A1 away from"`. Must be bound to a valid value; an empty or missing value raises an error. See the Enumerated / Constrained Values section. Supports expressions. |
| `depth` | string | count | No | `"1"` | Number of labware pieces to pick up from the top of the stack. Evaluated as an expression to an integer. Ignored when `moveEntireStack` is `true`. Engine tolerates omission (falls back to `stackDepth - 1`). |
| `moveEntireStack` | boolean | — | No | `false` | When `true`, picks up the entire stack regardless of `depth`. When `false` and `depth` is unbound, picks up all but the bottom piece (requires stack depth > 1). Engine tolerates omission (default `false`). |
| `gripperXOffset` | numeric | cm | No | *(unbound)* | Gripper X-axis offset override. Overrides position/labware-class grip offset when bound. Same unit convention as Move Labware and Move Pod. |
| `gripperYOffset` | numeric | cm | No | *(unbound)* | Gripper Y-axis offset override. |
| `gripperZOffset` | numeric | cm | No | *(unbound)* | Gripper Z-axis offset override. |
| `squeeze` | numeric | cm | No | *(unbound)* | Gripper squeeze (inward clamp distance) override. |
| `unsqueeze` | numeric | cm | No | *(unbound)* | Gripper unsqueeze (outward release distance) override. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

---

## Enumerated / Constrained Values

### `gripSide`

| Value | Meaning |
|-------|---------|
| `"A1 near"` | The gripper approaches at the deck position's nominal rotation; the labware's A1 corner faces the near (reference) side. |
| `"A1 away from"` | The gripper approaches rotated 180° (about the vertical axis) from `"A1 near"`, so the A1 corner faces the opposite side. Same grip geometry — only the orientation flips. |

Both values are *normal* grips: the gripper fingers close across the labware's **Y span**. (The wide-grip orientations, which close across the X span at a 90°/270° rotation, exist in the underlying enum but are **not** offered by Hold Labware — only these two values are valid here.)

The comparison is case-insensitive, so `"a1 near"` and `"A1 Near"` are accepted. A bound non-empty value that matches neither `"A1 near"` nor `"A1 away from"` raises an "invalid grip side" error. Both an unbound (missing) value and a bound empty string raise a "missing grip side" error; the step cannot distinguish the two cases.

### Stacking mode (derived from `moveEntireStack` and `depth`)

| `moveEntireStack` | `depth` bound? | Behavior |
|-------------------|----------------|----------|
| `true` | *(ignored)* | Picks up the entire stack. |
| `false` | Yes | Picks up exactly `depth` pieces from the top of the stack. |
| `false` | No (unbound) | Picks up all but the bottom piece (top stack-count minus one). Requires stack depth > 1. |

---

## Cross-Field Validation Rules

1. **Pod must have a gripper**: The named `pod` must be installed on the instrument and must have a gripper. An empty pod name raises a "pod must be specified" error; a non-existent pod raises a "pod not found" error. A pod that exists but has no gripper attached surfaces as a generic COM automation error from the gripper pick call.
2. **Source must be specified and contain labware**: `source` must be a non-empty string that resolves to a deck position containing at least one labware item. Empty string, non-existent position, and empty position each raise a distinct error.
3. **Grip side is mandatory**: `gripSide` must be bound and evaluate to `"A1 near"` or `"A1 away from"`. A missing value raises a "missing grip side" error; an unrecognized value raises an "invalid grip side" error.
4. **Leave-bottom requires stack depth > 1**: When `moveEntireStack` is `false` and `depth` is unbound, the computed depth is the stack height minus one. If the source has only one piece of labware, this yields zero and raises a "cannot leave bottom labware" error.
5. **`moveEntireStack` overrides `depth`**: When `moveEntireStack` is `true`, the `depth` key is entirely ignored and the full stack depth is used.
6. **Depth must be evaluable and not exceed the stack**: When `depth` is bound, its value is passed through expression evaluation. A value that cannot be evaluated to an integer is reported as a "bad labware depth" error. Requesting more than the available stack height is reported as a source-depth-vs-stack-height mismatch, and a resolved depth that reaches no labware raises an "unable to retrieve labware" error.
7. **Gripper-info override must exist on labware class**: If the source position has a `GripperInfoOverride` value, the labware class being gripped must have a matching key under its `GripperInfoOverrides` dictionary. A missing override raises a "missing movement information" error.
8. **Step-parameter offsets take final precedence**: If any of the five offset keys (`gripperXOffset`, `gripperYOffset`, `gripperZOffset`, `squeeze`, `unsqueeze`) is bound in the step's parameters, it overrides the corresponding value from position/labware-class gripper-info overrides.

---

## Structural Context

The Hold Labware step is a **container** with the following structural rules:

- **Has `subSteps`**: The step must contain a `subSteps` array.
- **Terminator**: The last element of `subSteps` must be an `"End"` step.
- **Allowed children**: Any step type may appear between the container's own
  parameters and the terminating End step. Typical children include
  liquid-handling steps (Span-8 Aspirate/Dispense, Multichannel
  Aspirate/Dispense) or other container steps. The step imposes no restrictions
  on child types, but it is not possible to have a Move Labware child step that
  uses the pod that is holding labware, as labware is already in the gripper.
- **Auto-release on End**: The gripper `Place` call occurs after all child steps have been enqueued, not in the End step itself. The End step is a structural marker only; the Hold Labware container's `Enqueue` method handles both the `Pick` (before children) and `Place` (after children).
- **A structural validation rule**: Per the JSON Import Structural Validation Spec, a Hold Labware step that does not end with an End step as its last child must be rejected.

---

## Canonical Examples

### Example 1: Hold a lid while dispensing into the lidded labware (i5 Span-8)

```json
{
  "stepType": "Hold Labware",
  "parameters": {
    "depth": "1",
    "gripSide": "A1 near",
    "moveEntireStack": false,
    "pod": "Pod1",
    "source": "P2"
  },
  "subSteps": [
    {
      "stepType": "Span-8 Dispense",
      "parameters": {
        "amount": "50",
        "aspirate": false,
        "autoSelectPrototype": true,
        "customHeight": false,
        "discardExcess": false,
        "dynamic?": true,
        "emptyTips": false,
        "firstWell": 1,
        "firstWellExpression": "",
        "height": 1.5,
        "heightFrom": 1,
        "liquidtype": "Tip Contents",
        "mandrelExpression": "",
        "operation": "Dispense",
        "overrideHeight": false,
        "pod": "Pod1",
        "podType": "Span8",
        "prototype": "S8 1000 Medium",
        "spacing": "1",
        "useExpression": false,
        "useProbes": {
          "_biomekType": "comArray",
          "arraySubtype": "boolean",
          "values": [
            true,
            true,
            true,
            true,
            true,
            true,
            true,
            true
          ]
        },
        "useWellExpression": false,
        "what": "BCFlat96",
        "where": "P2"
      }
    },
    {
      "stepType": "End",
      "parameters": {
        "caption": "End Holding Labware",
        "helpContextID": 141
      }
    }
  ]
}
```

### Example 3: Minimal — empty container (no child work steps)

```json
{
  "stepType": "Hold Labware",
  "parameters": {
    "depth": "1",
    "gripSide": "A1 near",
    "moveEntireStack": false,
    "pod": "Pod1",
    "source": "P7"
  },
  "subSteps": [
    {
      "stepType": "End",
      "parameters": {
        "caption": "End Holding Labware",
        "helpContextID": 141
      }
    }
  ]
}
```

---

## Common Mistakes

- **Expecting a `target`/`destination` parameter**: Hold Labware always returns labware to the same `source`. Use a separate Move Labware step for relocation.
- **`depth` as a numeric type instead of a string**: The default is `"1"` (string), and UI round-trip depends on the string form; the runtime tolerates a number but the method won't round-trip cleanly. Always author as a string (e.g., `"1"`, `"2"`).
- **Setting both `moveEntireStack: true` and a bound `depth`**: `moveEntireStack` wins unconditionally — `depth` is not consulted. Use one mode.
- **Binding an offset key to `0` instead of omitting it**: `0` is a bound override that displaces position/labware-class defaults; omit the key to inherit.
- **Leave-bottom mode with a single-item stack**: `moveEntireStack: false` + unbound `depth` computes (stack height minus one) = 0, raising a "cannot leave bottom labware" error. Use `moveEntireStack: true` for a single item.
