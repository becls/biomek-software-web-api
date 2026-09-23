# Next Labware

| Property | Value |
|----------|-------|
| stepType | `"Next Labware"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i7, i5 — **not** declared for i3 |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Structure/envelope: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Next Labware step cycles a named labware group (created by `Create Labware Group`) to
its next member. Each time it runs it removes the current labware from the working
`location` — returning it to its home position or off-deck device — brings the next unused
member of the group to `location` using `pod`/`gripSide` for gripper moves, and updates the
group's tracking global to the new position.

After a Next Labware step for a group with name `MyGroup`, other steps can
reference the current labware in the group's position using the expression `=MyGroup`.

Three booleans tune it:

- `checkUsed` — only cycle when the current labware is actually "used".
- `leaveOnDeck` — if the next member is already on deck, repoint to it instead of physically moving it.
- `breakOnFail` — when no unused member remains, signal the enclosing loop to break instead of erroring.

This is a **leaf** step and the cycle half of the
Create-Labware-Group / Next-Labware pair — it is meant to be placed inside a `Loop`, with
`breakOnFail` true so the loop ends when the group is exhausted.

The Next Labware step must come after a Create Group step.

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `name` | `expression-capable string` | — | Yes | `""` | Name of the labware group to cycle. **Note the key is `name`, not `groupName`** — `Create Labware Group` uses `groupName` for the same value. Trimmed and evaluated; empty or unknown group is an error. Group must be previously created with a Create Group step. |
| `location` | `expression-capable string` | — | Yes | `""` | Working deck position: the current labware is removed from here and the next member brought here. |
| `pod` | `string` | — | Yes | `""` | Name of the pod whose gripper performs physical moves (e.g. `"Pod1"`). **Always required**. |
| `gripSide` | `string (enum)` | — | Conditional | `""` (no runtime default; the `A1 near` fallback is print-only) | Grip side for gripper moves. Required whenever a gripper move occurs; in practice, always author it. |
| `checkUsed` | `boolean` | — | No | `true` | When true, only cycle if the current labware has been "used" (acted upon in a pipetting step); when false, always replace it — and in that case the search for the next member also skips members that were already retrieved once, so the group still advances. |
| `leaveOnDeck` | `boolean` | — | No | `true` | When true, if the next member is already on deck leave it in place to be used where it is (no gripper move); when false, move it to `location` anyway. It also decides what happens when the outgoing labware's home position is occupied: true leaves that labware where it is and points the group at the next member in its own position; false raises the rule 6 error. |
| `breakOnFail` | `boolean` | — | No | `true` | When true and no unused member remains, an enclosing loop swallows the failure and breaks; when false, continue (the raised error propagates). |

Universal base keys (`caption`, `dynamic?`, …) — see
[Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).

## Enumerated / Constrained Values

### `gripSide`

| Value | Meaning |
|-------|---------|
| `"A1 near"` | A1 corner near the pod. |
| `"A1 away from"` | A1 corner away from the pod. |

## Cross-Field Validation Rules

Enqueue-time validation rules:

1. `pod` **must** name a pod that exists on the instrument — otherwise the step fails as an unknown-pod error.
2. `name` (the group) **must** be non-empty — otherwise the step fails as an
   empty-group-name error.
3. The group **must** exist in the deck's `LabwareGroups` — otherwise the step
   fails as a group-not-found error. Create a group with the Create Group step before
   using Next Labware.
4. Labware sitting at `location` **must** belong to the group — otherwise the step fails as a labware-not-in-group error.
5. The outgoing labware **must** have a recorded home — otherwise the step fails as a no-home error.
6. When the home position is occupied and `leaveOnDeck` is false, the move is blocked as a return-blocked error.
7. Bringing in the next member with a missing destination fails as a destination-not-found error.
8. When no unused member remains, the step fails as a no-unused-member error. If `breakOnFail` is true, an enclosing loop breaks instead of failing.
9. If the outgoing labware's home is not a deck position (it came from an off-deck storage device) and no suitable off-deck destination is free, the step fails as a no-off-deck-destination error.
10. If the outgoing labware had to be left in place (its home was occupied and `leaveOnDeck` is true) and the next member lives on a storage device, the step fails as a cannot-move-next-labware error.

## Structural Context

This step is a **leaf** and
does not support `subSteps`. Include no `subSteps`.

**Pairing.** Next Labware is meaningless on its own — it requires a group created earlier by
`Create Labware Group` (see [Create Labware Group](72-Create-Labware-Group.md)) and is
placed inside a `Loop` with `breakOnFail = true` so the loop terminates when
the group is exhausted (the exhaustion error is absorbed and the loop exits cleanly).

## Canonical Examples

Cycle a group inside a loop, moving each member to the working position:

```json
{
  "stepType": "Next Labware",
  "parameters": {
    "name": "SourcePlates",
    "location": "P5",
    "pod": "Pod1",
    "gripSide": "A1 near",
    "checkUsed": false,
    "leaveOnDeck": false,
    "breakOnFail": true
  }
}
```

Repoint at the next member instead of fetching it (`leaveOnDeck` true). This suppresses only the
*incoming* move — returning the labware currently at `location` to its home can still emit a
gripper move. `pod` is still supplied — it is resolved on every execution:

```json
{
  "stepType": "Next Labware",
  "parameters": {
    "name": "SourcePlates",
    "location": "P5",
    "pod": "Pod1",
    "gripSide": "A1 near",
    "checkUsed": false,
    "leaveOnDeck": true,
    "breakOnFail": true
  }
}
```

Typical loop wiring (paired with Create Labware Group, not shown):

```json
{
  "stepType": "Loop",
  "parameters": { "variable": "", "start": "1", "end": "999", "increment": "1" },
  "subSteps": [
    {
      "stepType": "Next Labware",
      "parameters": { "name": "SourcePlates", "location": "P5", "pod": "Pod1", "gripSide": "A1 near", "breakOnFail": true }
    },
    { "stepType": "Transfer", "parameters": {} },
    { "stepType": "End", "parameters": {} }
  ]
}
```

## Common Mistakes

- **Using `groupName` instead of `name`**: this step reads the group under the `name` key,
  unlike `Create Labware Group` which uses `groupName`. Using `groupName` here silently
  leaves the group unspecified.
- **No enclosing loop with `breakOnFail`**: without the loop, group exhaustion surfaces as a
  runtime error; with `breakOnFail = true` inside a `Loop` it's absorbed as a clean exit.
- **`checkUsed = true` on labware never marked used**: if the current labware isn't flagged
  "used" by a preceding pipetting step, the step re-pins the group's tracking global to the
  current `location` and returns without selecting a new member — producing a loop that never
  advances.
- **Omitting `pod`**: the pod is resolved on every execution, before the move-vs-no-move
  decision, so an absent or unknown `pod` fails as an unknown-pod error even in a repoint-only
  configuration.
- **Expecting `leaveOnDeck` to suppress all movement**: it governs only the *incoming* member.
  Returning the labware currently at `location` to its home still emits a gripper move.
- **Assuming i3 support**: this step is declared for i5 and i7 only. A method containing it still opens on an i3,
  but enqueuing fails as a step-not-compatible error.
