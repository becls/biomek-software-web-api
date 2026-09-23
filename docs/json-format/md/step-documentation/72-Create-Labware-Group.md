# Create Labware Group

> **Reference manual:** this step appears as *Create Group* (Chapter 30); the JSON `stepType` is `"Create Labware Group"`.

| Property | Value |
|----------|-------|
| stepType | `"Create Labware Group"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i7, i5 — **not** declared for i3 |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Structure/envelope: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Create Labware Group step tags a set of labware as a named, ordered group so a later
`Next Labware` step can cycle through them. The current labware can then be
referenced using an expression `=group` where `group` is a value defined in `groupName`.
If `moveFirstLW` is true it immediately fetches the first member to `location` using `pod`
and `gripSide`. This is a **leaf** step and is the create
half of the Create-Labware-Group / Next-Labware pair; it is meant to be followed by a `Loop`
containing a `Next Labware`.

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `positions` | array | — | Yes | *(none — must be bound)* | Ordered list of group members. Each element is either a **string** deck-position name (expression-capable — `"=Pos"` resolves through a method variable), or a nested dictionary with `device`/`stack`/`endStack` for a storage-device stack (see below). The step fails as a not-configured error if this key is absent. |
| `groupName` | `expression-capable string` | — | Yes | `""` | Name of the group being created. Trimmed, then evaluated as an expression; must be a valid identifier (see rule 3). Paired with `Next Labware`, which reads the same group under the `name` key (not `groupName`). |
| `moveFirstLW` | `boolean` | — | No | `false` | When true, immediately move the first group member to `location`. |
| `location` | `expression-capable string` | — | Conditional | `""` | Destination deck position for the first labware when `moveFirstLW` is true. Ignored otherwise. |
| `pod` | `string` | — | Conditional | `""` | Pod/gripper used for the initial move when `moveFirstLW` is true (e.g. `"Pod1"`). |
| `gripSide` | `string (enum)` | — | Conditional | `""` | Grip side for the initial move when `moveFirstLW` is true. Must be `"A1 near"` or `"A1 away from"`. See enum below. Same values and semantics as the `gripSide` key on the other labware-handling steps (`Move Labware`, `Hold Labware`, `Next Labware`). |

Universal base keys (`caption`, `dynamic?`, …) — see
[Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).

### `positions` member element forms

| Form | Shape | Meaning |
|------|-------|---------|
| On-deck position | JSON string, e.g. `"P1"` | Name of a deck position; the labware currently there joins the group. A name that is not a deck position, or a position holding no labware, is skipped silently — it still counts as a member of the group, but nothing is tagged. |
| Storage-device stack | nested dictionary `{ "device": "...", "stack": "...", "endStack": "..." }` | A stack on a SILAS storage device. Each of `device`/`stack`/`endStack` is an expression string; every non-null labware in `stack` joins the group and is given `GoToIndex = endStack`. |

## Enumerated / Constrained Values

### `gripSide`

| Value | Meaning |
|-------|---------|
| `"A1 near"` | A1 corner held near the pod. |
| `"A1 away from"` | A1 corner held away from the pod. |

## Cross-Field Validation Rules

Enqueue-time validation rules:

1. `positions` **must** be bound — otherwise the step fails as a not-configured error.
2. `groupName` (trimmed, evaluated) **must** be non-empty — otherwise the step fails as an empty-group-name error.
3. `groupName` **must** be a valid identifier — otherwise the step fails as an invalid-group-name error.
4. A storage-device member's `device` **must** exist — otherwise the step fails as a device-not-found error.
5. The member device **must** use the `Stacks` storage model — otherwise the step fails as an unsupported-device error.
6. The named `stack` and `endStack` **must** exist on the device — otherwise the step fails as a stack-not-present error.
7. A member stack **must not** contain nested labware stacks — otherwise the step fails as a nested-stacks-not-supported error.
8. Every expression-capable field — `groupName`, each `positions` entry, and a member's
   `device`/`stack`/`endStack` — **must** evaluate; an unresolvable expression fails as an evaluation error naming the field.

> If `moveFirstLW` is true, the initial fetch runs the `Next Labware` engine and can additionally raise any of that
> step's errors (see [Next Labware](73-Next-Labware.md)).

## Structural Context

This step is a **leaf** and
does not support `subSteps`. Include no `subSteps`.

## Canonical Examples

Create a group from four deck positions (no immediate move):

```json
{
  "stepType": "Create Labware Group",
  "parameters": {
    "groupName": "SourcePlates",
    "positions": ["P1", "P2", "P3", "P4"],
    "moveFirstLW": false
  }
}
```

Create a group and bring the first member onto the working position:

```json
{
  "stepType": "Create Labware Group",
  "parameters": {
    "groupName": "SourcePlates",
    "positions": ["P1", "P2", "P3"],
    "moveFirstLW": true,
    "location": "P5",
    "pod": "Pod1",
    "gripSide": "A1 near"
  }
}
```

Create a group from a storage-device stack:

```json
{
  "stepType": "Create Labware Group",
  "parameters": {
    "groupName": "StackedPlates",
    "positions": [
      { "device": "Stacker1", "stack": "In", "endStack": "Out" }
    ],
    "moveFirstLW": false
  }
}
```

## Common Mistakes

- **Use `groupName` here but `name` in `Next Labware`**: the group-name key differs
  between the two steps — `groupName` to create, `name` to cycle. Keep the string value the
  same across both.
- **`moveFirstLW` without a valid `pod`**: the initial fetch resolves `pod` before it does
  anything else, so a blank or unknown pod fails as an unknown-pod error rather than a
  parameter-name error. Problems with `location`/`gripSide` surface as the paired
  `Next Labware` errors when the fetch actually has to move labware.
- **Nested labware stacks in a storage-device member**: not supported (rule 7). Flatten
  first or the group creation aborts partway.
- **Assuming i3 support**: this step is declared for i5 and i7 only. A method containing it still opens on an i3,
  but enqueuing fails as a step-not-compatible error.
