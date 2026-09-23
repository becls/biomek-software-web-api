# Group

| Property | Value |
|----------|-------|
| stepType | `"Group"` |
| Category | Free-form container |
| Terminator | `"End"` (caption: "End Group") |
| Compatible Hardware | All |

## Behavior Summary

The Group step is a free-form container. At runtime it enqueues its child steps in order and does nothing else — no looping, no branching, no scope. The step tree renders it with an optional description; child steps are indented within it until the "End Group" terminator.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `description` | `string` | — | No | `""` | Description text for the group. Not expression-evaluated. Editor-written; not read at runtime. Every editor-produced step carries this key, so exports always include it — emit `""` when the group has no description, and **always author this key.** |
| `comment` | `string` | — | No | — | Optional comment text for the group. Not expression-evaluated. Editor-written; not read at runtime. Preserve when round-tripping. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

If any additional keys appear on a `Group` step in an exported method, they do not impact the behavior of the Group step. Preserve them verbatim for round-tripping.

## Enumerated / Constrained Values

None.

## Cross-Field Validation Rules

None. The group is a simple container with no validation of its own.

## Structural Context

This step is a **free-form container**. It uses `subSteps` and is terminated by an auto-generated `"End"` step (whose caption is `"End Group"`). **The terminator's `stepType` is `"End"`, not `"End Group"`** — `"End Group"` is only the display caption; there is no `"End Group"` step type registered, so serializing `"stepType": "End Group"` fails import.

```jsonc
{
  "stepType": "Group",
  "parameters": { ... },
  "subSteps": [
    { ... child steps ... },
    { "stepType": "End", "parameters": { "caption": "End Group" } }
  ]
}
```

The terminator must always be the last element in `subSteps`.

## Canonical Examples

Group with child steps:

```json
{
  "stepType": "Group",
  "parameters": {
    "description": "Reagent addition"
  },
  "subSteps": [
    {
      "stepType": "Fixed-8 Load Tips",
      "parameters": { "tips": "P1", "mode": "NumberOfTips", "numberOfTips": "8" }
    },
    {
      "stepType": "Fixed-8 Aspirate",
      "parameters": { "pod": "Pod1", "operation": "Aspirate", "where": "P3", "amount": "100", "firstWell": 1, "liquidType": "Well Contents", "autoSelectPrototype": false, "prototype": "F8 Medium" }
    },
    {
      "stepType": "Fixed-8 Dispense",
      "parameters": { "pod": "Pod1", "operation": "Dispense", "where": "P4", "amount": "100", "firstWell": 1, "liquidType": "Tip Contents", "autoSelectPrototype": false, "prototype": "F8 Medium" }
    },
    {
      "stepType": "Fixed-8 Unload Tips",
      "parameters": { "tipDestination": "<Any Trash>" }
    },
    { "stepType": "End", "parameters": { "caption": "End Group" } }
  ]
}
```

Empty group (description only):

```json
{
  "stepType": "Group",
  "parameters": {
    "description": "Placeholder for future steps"
  },
  "subSteps": [
    { "stepType": "End", "parameters": { "caption": "End Group" } }
  ]
}
```

## Common Mistakes

- **Serializing the terminator as `"stepType": "End Group"`**: `"End Group"` is only the display caption; the registered stepType is `"End"`. Emitting `"End Group"` fails import.
- **Expecting control flow**: Group does not loop or branch; it just runs children sequentially. Use `If`/`Loop`/`Worklist` instead.
