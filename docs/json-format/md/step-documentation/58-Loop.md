# Loop

| Property | Value |
|----------|-------|
| stepType | `"Loop"` |
| Category | Container (free-form) |
| Terminator | `"End"` (caption: "End Loop") |
| Compatible Hardware | All |

## Behavior Summary

The Loop step repeats its child steps a computed number of times based on a start value, end value, and increment. At runtime it evaluates the `start`, `end`, and `increment` parameters (which may be expressions), calculates the iteration count as `floor((end - start) / increment) + 1`, and executes its children once per iteration. If a `variable` name is provided, each iteration binds that variable to the current loop value (`start + increment × index`) for use by child steps. If no variable is provided, iterations are labeled "Loop pass #N". The Loop step validates that the increment direction is consistent with the start-to-end direction — a positive increment with start < end, or a negative increment with start > end. When start equals end, exactly one iteration executes regardless of increment. The loop variable is scoped to the child steps. A [`Break`](49-Break.md) step inside the loop ends iteration early; `Break`'s `levels` counts enclosing **Loop** steps only, not intervening containers. If a child step raises an error, the Loop re-reports the error tagged with the 1-based iteration number.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `variable` | `string` | — | No | `""` | Name of the loop variable. When non-empty, each iteration binds this name to the current loop value, so child steps can read it as `=Name` in any expression-capable field. When empty, child steps cannot reference the loop variable. Use a valid script identifier — no spaces, no punctuation such as `.` `,` `&`, no leading digit. An empty `variable` is valid. Omitting the key entirely is equally valid. |
| `start` | `expression-capable string` | — | Yes | `"1"` *(editor default only — JSON import does not apply it; always author the key)* | Starting value of the loop counter. Evaluated as a double. If prefixed with `=`, evaluated as a script expression. |
| `end` | `expression-capable string` | — | Yes | `"1"` *(editor default only — JSON import does not apply it; always author the key)* | Ending value of the loop counter. The loop iterates while the counter has not passed this value. Evaluated as a double. If prefixed with `=`, evaluated as a script expression. |
| `increment` | `expression-capable string` | — | Yes | `"1"` *(editor default only — JSON import does not apply it; always author the key)* | Step value added to the counter each iteration. Evaluated as a double. If prefixed with `=`, evaluated as a script expression. |

### Internal Keys

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, `disabled`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

## Enumerated / Constrained Values

No enumerated value constraints. The three counter parameters (`start`, `end`, `increment`) accept numeric literals or expression strings prefixed with `=`.

## Cross-Field Validation Rules

1. When `start` < `end`, `increment` **must** be positive (> 0); otherwise enqueue fails.
2. When `start` > `end`, `increment` **must** be negative (< 0); otherwise enqueue fails.
3. When `start` == `end`, exactly one iteration executes regardless of `increment` value (including 0).
4. `start`, `end`, and `increment` **must** all be present and non-empty. An omitted key fails at Run on the missing-key lookup; an empty value cannot be converted to a number and so fails at enqueue; a printed method also flags this in place of the loop line.
5. `variable`, when supplied, must be a valid script identifier. An invalid name is not rejected at enqueue, but fails when a child step references it.

## Structural Context

The Loop step is a **free-form container**. It requires one child:

- **Trailing anchor**: an `"End"` step (caption `"End Loop"`) is required as the container's last child. Author it as the last element of `subSteps`.

**Constraints**:

- `subSteps` must be present and contain at least the `"End"` terminator as the last element.
- Any number of operational steps may be placed before the `"End"` terminator.
- A Loop with only `"End"` (no operational steps) is structurally valid but does nothing.
- Child steps reference the loop variable via `=VariableName` syntax.

## Canonical Examples

Simple loop repeating 3 times with a named variable:

```json
{
  "stepType": "Loop",
  "parameters": {
    "variable": "Count",
    "start": "1",
    "end": "3",
    "increment": "1"
  },
  "subSteps": [
    { "stepType": "Transfer", "parameters": {} },
    { "stepType": "End", "parameters": { "caption": "End Loop" } }
  ]
}
```

Loop with an expression bound, whose child consumes the loop variable — the child aspirates from a first well that advances one column per pass, starting at well 2. Only the variable-consuming keys are shown; the aspirate's other required keys are omitted for brevity (see [Fixed-8 Aspirate](36-Fixed-8-Aspirate.md)):

```json
{
  "stepType": "Loop",
  "parameters": {
    "variable": "column",
    "start": "2",
    "end": "=NumColumns",
    "increment": "1"
  },
  "subSteps": [
    {
      "stepType": "Fixed-8 Aspirate",
      "parameters": {
        "useWellExpression": true,
        "firstWellExpression": "=column"
      }
    },
    { "stepType": "End", "parameters": { "caption": "End Loop" } }
  ]
}
```

Countdown loop (negative increment):

```json
{
  "stepType": "Loop",
  "parameters": {
    "variable": "Step",
    "start": "10",
    "end": "1",
    "increment": "-1"
  },
  "subSteps": [
    { "stepType": "Transfer", "parameters": {} },
    { "stepType": "End", "parameters": { "caption": "End Loop" } }
  ]
}
```

## Common Mistakes

- **Omitting `start`, `end`, or `increment`**: the editor default of `"1"` does *not* survive JSON import — import replaces the step's dictionary with exactly the keys you author. Enqueue reads each key with a throwing accessor, so an omitted key imports fine and then fails at Run. Always emit all three.
- **Emitting `start`/`end`/`increment` as JSON numbers**: the editor always writes these as strings, and expressions (`"=NumCols"`) can only be strings. Author `"1"`, not `1`, so the method round-trips identically to an editor-saved export.
- **Forgetting the `"End"` terminator**: The Loop is a free-form container and must end with `"End"`.
- **Confusing `end` (loop bound) with `"End"` (terminator step)**: The `end` parameter is the loop counter bound; `"End"` is the structural terminator step type that must be the last child.
- **Assuming iteration count equals end value**: Iteration count is `floor((end - start) / increment) + 1`, not `end`. For example, start=1, end=100, increment=25 gives 4 iterations (1, 26, 51, 76), not 100.
- **Multiplying the loop variable by the probe count in a well expression**: when a Loop sweeps plate columns for a Fixed-8 or Span-8 step, pass the loop variable straight into `firstWellExpression` — `"=column"`, as in the second example above — not `"=1+(column-1)*8"`. Well numbering is row-major, so the top well of column *c* is simply well *3*; multiplying by 8 assumes column-major numbering and lands the first tip mid-plate — the wrong wells, and a run-time tip-fit failure as soon as fewer than eight rows remain below it. See [Probe & Mandrel Selection](../concept-guides/03-probe-and-mandrel-selection.md).
