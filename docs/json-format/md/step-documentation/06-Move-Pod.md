# Move Pod

| Property | Value |
|----------|-------|
| stepType | `"Move Pod"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | Any |

---

## Behavior Summary

The Move Pod step moves a pod to a specified deck position without performing
any pipetting. This step may be used to move the pod out of the way before manual
operations on the deck (e.g., adding or removing labware) or before a Pause
step.

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | string | — | Yes | `""` | Name of the pod to move (e.g., `"Pod1"`, `"Pod2"`). Must resolve to a pod installed on the instrument. |
| `location` | string | — | Yes | `""` | Deck position or labware name to move the pod over (e.g., `"P3"`, `"P13"`). Must resolve to a valid deck position on the instrument. Supports expressions. |
| `xOffset` | numeric | cm | **Yes** | `0.0` | Left to right offset in centimeters from the deck position origin. Added to the resolved labware origin X. **Must be present** (as `0.0` if not offsetting); enqueue reads via dictionary indexer and raises a "could not find XOffset" system error when omitted. |
| `yOffset` | numeric | cm | **Yes** | `0.0` | Front to back offset in centimeters from the deck position origin. Added to the resolved labware origin Y. **Must be present** (as `0.0` if not offsetting); enqueue reads via dictionary indexer and raises a "could not find YOffset" system error when omitted. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

---

## Enumerated / Constrained Values

### `pod`

The `pod` value must name one of the pods installed on the instrument. Valid values are `"Pod1"` (default for single-pod instruments) and `"Pod2"` (second pod on dual-pod Biomek i7 systems).

### `location`

The `location` value must name a valid deck position from the instrument's deck layout. Valid position names depend on the instrument model and deck configuration — consult the installed instrument's deck editor for the actual position names (they vary by chassis model and ALP configuration). This value may also be an expression that evaluates to a valid deck position name.

### `xOffset`, `yOffset`

Unconstrained numeric values in **centimeters** (not millimeters). Zero is the default and positions the pod directly over the deck position's labware origin. Non-zero offsets shift the final position by the specified amount.

---

## Cross-Field Validation Rules

1. **Pod must be valid**: The `pod` string must resolve to a pod installed on the instrument.
2. **Location must be valid**: The `location` string must resolve to a valid deck position on the instrument. An unresolvable location causes an error at enqueue time.
3. **Pod and Location are both required for execution**: If either `pod` or `location` is empty at print time (the method report and printed output), the step prints a not-configured placeholder. The design-caption is left blank in that case.
4. **Offsets are independent**: `xOffset` and `yOffset` operate independently and do not influence each other. Both keys must be present at enqueue; author `0.0` explicitly when not offsetting (`0.0` is the conventional value, not a runtime default — see the Parameters table).
5. **Labware alignment is automatic**: When labware is present at the target location, positional adjustment is computed internally from labware class geometry. The user-supplied offsets are added on top of this automatic alignment — they do not replace it.
6. **Resolved destination must be within the pod's travel range**: The final X/Y (deck position origin plus `xOffset`/`yOffset` plus automatic alignment) must lie inside the pod's physical travel limits. A destination outside the limit fails at enqueue with a "destination outside travel range" error. Large offsets are the common trigger.
7. **Pod moves are path-planned at runtime**: The approach is routed by the runtime path planner, which can fail even for an in-range destination when no collision-free path exists.

---

## Structural Context

This step is a **leaf** and does not support `subSteps`. Never include a `subSteps` array on a Move Pod step.

The Move Pod step may appear anywhere in the body of a method (between Start and Finish).

---

## Canonical Examples

### Minimal Move Pod (defaults — most common case)

Move Pod1 to deck position P3 with no offset:

```json
{
  "stepType": "Move Pod",
  "parameters": {
    "pod": "Pod1",
    "location": "P3",
    "xOffset": 0.0,
    "yOffset": 0.0
  }
}
```

### Move Pod with offsets

Move Pod2 to deck position P13, shifted 1.5 cm in X and −0.5 cm in Y:

```json
{
  "stepType": "Move Pod",
  "parameters": {
    "pod": "Pod2",
    "location": "P13",
    "xOffset": 1.5,
    "yOffset": -0.5
  }
}
```

### Move Pod before a Pause step

A common pattern: move the pod out of the way, then pause for manual intervention:

```json
[
  {
    "stepType": "Move Pod",
    "parameters": {
      "pod": "Pod1",
      "location": "P1",
      "xOffset": 0.0,
      "yOffset": 0.0
    }
  },
  {
    "stepType": "Pause",
    "parameters": {
      "location": "",
      "message": "Add reagent plate to P12, then click Resume.",
      "mode": "PromptedGlobal",
      "time": "0"
    }
  }
]
```

---

## Common Mistakes

- **Omitting `xOffset` or `yOffset`** — Both are read via a throwing indexer; missing keys throw a "could not find XOffset/YOffset" system error at enqueue. Use `0.0` when not offsetting.
- **Confusing Move Pod with Move Labware** — Move Pod repositions the pod; no labware is touched. Move Labware physically transfers labware using a gripper.
- **Assuming offsets replace labware alignment** — `xOffset`/`yOffset` are *added on top of* the automatic labware-origin and well-alignment adjustments, not substituted for them.
- **Offsets that push the pod past its travel limit** — Large offsets or corner destinations fail at enqueue with a "destination outside travel range" error.
- **Forgetting Move Pod before a Pause that requires deck access** — Without it, the pod may block the area the user needs to reach.
