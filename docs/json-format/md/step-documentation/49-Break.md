# Break

| Property | Value |
|----------|-------|
| stepType | `"Break"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | All |

## Behavior Summary

The Break step terminates execution of one or more enclosing loops. When executed, it exits the specified number of enclosing loop iterations. A level of 1 breaks out of the innermost loop; a level of 2 breaks out of two nested loops, etc. The Break step must be placed inside a Loop (or nested Loops) to have any effect.

**`levels` counts enclosing Loop steps only.** Only the Loop step decrements the break counter. Non-Loop containers (`If`, `Group`, `Just In Time`, `Let`, …) do
not touch the break counter, so they are **not** counted when tallying `levels`.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `levels` | `expression-capable string` | — | Yes | `"1"` | Number of enclosing **Loop steps** to break out of (see §Behavior Summary — `levels` counts Loops only, not intermediate containers such as If/Group/Just In Time). Evaluated as an integer. A value of `"1"` exits the innermost Loop; `"2"` exits two nested Loops. |

## Enumerated / Constrained Values

No enumerated constraints. `levels` accepts any expression that evaluates to a positive integer.

## Cross-Field Validation Rules

1. `levels` **must** be bound in the dictionary; otherwise Enqueue fails.
2. `levels` **must** evaluate to a valid positive integer.
3. `levels` must not exceed the actual nesting depth of enclosing loops, and the Break must be inside at least one loop.

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

Break out of the innermost loop (default):

```json
{
  "stepType": "Break",
  "parameters": {
    "levels": "1"
  }
}
```

Break out of two nested loops:

```json
{
  "stepType": "Break",
  "parameters": {
    "levels": "2"
  }
}
```

Break using an expression:

```json
{
  "stepType": "Break",
  "parameters": {
    "levels": "=BreakDepth"
  }
}
```

## Common Mistakes

- **Omitting `levels`**: the step's built-in default does *not* apply during JSON import. Always emit the key.
- **`levels` exceeding actual loop nesting depth**: count the *enclosing* Loops, not the containers between them.
- **Counting non-Loop containers as levels**: `If`, `Group`, `Just In Time`, `Let`, etc. do **not** consume a break level. A Break inside `Loop → If → Loop → If → (here)` needs `levels: 2` to escape both Loops, not 4.
