# Just In Time

| Property | Value |
|----------|-------|
| stepType | `"Just In Time"` |
| Category | Free-form container |
| Terminator | `"End"` (caption: "End Just In Time") |
| Compatible Hardware | All |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Envelope / structural rules: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Just In Time step is a free-form container that wraps its child steps in a **queue-engine JIT block**, so their actions are scheduled "just in time" — as late as the schedule allows, instead of as early as their dependencies and resources permit. Children are still enqueued in order; only the timing of their actions changes. Unlike `Group`, which only organizes steps, Just In Time changes *when* the enclosed actions run; substituting `Group` changes the run schedule. Use it when the enclosed actions should happen as close as possible to the moment they are needed — for example, wrapping an aspirate/dispense pair so reagent is not drawn until the dispense is ready. Biomek always writes `slack` as `0` (see the parameter note).

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `slack` | number | seconds | **Yes** | `0` | Scheduling slop for the JIT block, consumed at enqueue. Biomek always writes `0` — the editor exposes no field for it — so author `0`. Required at enqueue: the key must be present, or Run fails on the missing-key lookup (see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md#0-what-required-means-in-the-parameters-table) §0). |

Universal base keys (`caption`, `isPreconfigured`, `dynamic?`, `disabled`, …)
— see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md); all are optional
editor/serializer bookkeeping.

## Enumerated / Constrained Values

`slack` must be a non-negative number of seconds. The queue engine rejects NaN, negative values, and values larger than 1e+8 seconds. Author the JSON number `0`.

## Cross-Field Validation Rules

None. The step itself performs no enqueue-time validation of its parameters beyond the `slack` range check the queue engine applies (see Enumerated / Constrained Values). Structural validity is governed only by the container rules below.

## Structural Context

Just In Time is a **free-form container**. Its final substep must be an `"End"` terminator captioned `"End Just In Time"`.

**Constraints** (per [Method JSON Structure](../Method-JSON-Structure.md)):

- `subSteps` must be present and must end with the `"End"` terminator as the last element.
- Any number of legal operational steps may be placed before the terminator.
- The terminator's serialized `stepType` is `"End"`; its caption is `"End Just In Time"`.

## Canonical Example

Defer an aspirate so the reagent is not drawn until its dispense is ready:

```json
{
  "stepType": "Just In Time",
  "parameters": { "slack": 0 },
  "subSteps": [
    {
      "stepType": "Fixed-8 Aspirate",
      "parameters": {
        "pod": "Pod1",
        "where": "P3",
        "operation": "Aspirate",
        "amount": "100",
        "firstWell": 1,
        "liquidType": "Well Contents",
        "autoSelectPrototype": false,
        "prototype": "F8 Medium"
      }
    },
    {
      "stepType": "Fixed-8 Dispense",
      "parameters": {
        "pod": "Pod1",
        "where": "P4",
        "operation": "Dispense",
        "amount": "100",
        "firstWell": 1,
        "liquidType": "Tip Contents",
        "autoSelectPrototype": false,
        "prototype": "F8 Medium"
      }
    },
    { "stepType": "End", "parameters": { "caption": "End Just In Time" } }
  ]
}
```

The wrapped children keep their own required keys — see
[Fixed-8 Aspirate](36-Fixed-8-Aspirate.md) and
[Fixed-8 Dispense](37-Fixed-8-Dispense.md). Load and unload tips outside the block;
any legal operational steps may be wrapped.

## Common Mistakes

- **Serializing the terminator as `"stepType": "End Just In Time"`:** `"End Just In Time"` is only the display caption; the registered step type is `"End"`. Emitting the caption as a step type fails import.
- **Omitting `slack` entirely:** the key is required at enqueue, so a missing key fails at Run. Author `0`.
- **Giving empty `parameters` to the wrapped child steps:** Just In Time does not relax its children's requirements — each child still needs its own required keys, or enqueue fails on that child.
- **Substituting `Group`:** `Group` only organizes steps; it does not defer them. Wrapping the same children in `Group` changes the run schedule.
