# Next Item

| Property | Value |
|----------|-------|
| stepType | `"Next Item"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | All |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Structure/envelope: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Next Item step advances a global variable to the next value in a list. At enqueue it
evaluates `variable` to a variable name, reads that global's current value, evaluates
`values` to an array (a bare string is wrapped via `=Array(...)`), finds the current value
in the list (case-insensitively), and assigns the following value to the global. When the
current value is the last one in the list — or the list itself is empty — behavior is
governed by `onDone`: `"Break"` breaks out of one enclosing loop, `"SetTo"` assigns the
evaluated `setTo` expression, and `"StartOver"` wraps back to the first value.

Two cases bypass `onDone` entirely and assign the **first** value in the list: the global
being unset or empty, and the global's current value not appearing in the list at all. The
second case is silent — a typo in the value, or a value written by another step, restarts
the sequence instead of raising.

It is the list-driven counterpart to a [`Loop`](58-Loop.md): place it inside a `Loop` whose
body consumes `variable`, and use `onDone = "Break"` to end the loop when the list is
exhausted. This is a **leaf** step; it takes no child steps.

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `variable` | `expression-capable string` | — | Yes | `""` | Name of the global variable to advance. Evaluated as an expression to a variable name. |
| `values` | `expression-capable string` | — | Yes | `""` | The value list. Without a leading `=` the string is taken literally and then wrapped as `=Array(<literal>)`, so `"P1","P2","P3"` and `=Array("P1","P2","P3")` are equivalent — the bare comma-separated form is the standard authored form. With a leading `=` it must evaluate to an array (or to a single string that `Array(...)` can wrap). |
| `onDone` | `string (enum)` | — | Yes | `"Break"` | What to do when the current value is the last one in the list (or the list is empty). One of `"Break"`, `"SetTo"`, `"StartOver"`. Read only when the list runs out, so omitting it fails at that moment rather than on every pass; always emit it. |
| `setTo` | `expression-capable string` | — | Conditional | `""` | Value assigned to `variable` when `onDone = "SetTo"` and the list is exhausted. Evaluated as an expression. Ignored for other `onDone` values. |
| `name` | `string` | — | No | — | Optional display alias. When bound, the step caption becomes `"Next <Name>"` instead of `"Next Item"`. Cosmetic only — no runtime effect. Not required; omitted in normal authored steps. |

Universal base keys (`caption`, `dynamic?`, …) — see
[Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md). Emit `variable`, `values`, `onDone` and `setTo` on every Next Item step; `caption` is optional.

## Enumerated / Constrained Values

### `onDone`

| Value | Meaning when the list is exhausted |
|-------|-------------------------------------|
| `"Break"` | Break out of one enclosing `Loop`. The global is **not** reassigned on this path — it keeps the last list value. |
| `"SetTo"` | Set `variable` to the evaluated `setTo` expression. |
| `"StartOver"` | Wrap back to the first value in the list. Requires at least **two** values (see rule 5). |

`onDone` is matched **case-sensitively**. Be sure to use the given correct capitalization.

On the `"Break"` path the step breaks before assigning, so `variable` is left holding the
last value in the list. Re-entering the loop with that value
still set breaks again immediately — reset the global with a
[`Set Global`](69-Set-Global.md) step before the loop if the loop can run more than once.

## Cross-Field Validation Rules

1. `variable` **must** be bound.
2. `values` **must** evaluate to an array, or to a single string that `Array(...)` can wrap.
3. Every element of `values` **must** be convertible to a string. Numeric elements are converted (`1` becomes `"1"`); an element that cannot be converted fails at enqueue (N is 1-based within the list).
4. The current value **must** match at most one entry — comparison is case-insensitive. If it matches two (e.g. `"A"` and `"a"`), the step fails at enqueue. Duplicates elsewhere in the list are not detected, so keep the whole list unique anyway.
5. `onDone = "StartOver"` requires at least **two** values. With an empty list — or a single-value list — the step fails at enqueue (misleading in the single-value case).

> Matching the current value against the list is case-insensitive, and it is a **string** comparison — `1` and `01` are different values. If the
> current global value is empty/unset, or is not in the list at all, the step assigns the
> **first** list value without consulting `onDone`.

## Structural Context

This step is a **leaf** and does
not support `subSteps`. Include no `subSteps` (or an empty array).

**Valid placement.** Next Item is not itself a container and its variable-advance works
anywhere, but `onDone = "Break"` requires an enclosing [`Loop`](58-Loop.md) — only `Loop`
consumes a break level. Other containers — `Worklist`,
`If`, `Group`, `Let`, `Just In Time` — do **not** absorb the break; a Next Item `"Break"`
inside a [`Worklist`](55-Worklist.md) with no outer `Loop` aborts the Worklist with a
pass error rather than ending it cleanly.

With no enclosing `Loop` at all, the break is uncaught and surfaces as a run-time error. See also [Break](49-Break.md)
for the break protocol, and [Next Labware](73-Next-Labware.md) for the
labware-group analog of this step.

## Canonical Examples

Bare comma-separated list — the standard authored form. Quote string values; numbers
need no quotes:

```json
{
  "stepType": "Next Item",
  "parameters": {
    "variable": "CurrentPlate",
    "values": "\"plate1\",\"plate2\",\"plate3\",\"plate4\"",
    "onDone": "Break",
    "setTo": ""
  }
}
```

The same thing written as an explicit expression — equivalent, because a bare `values`
string is wrapped as `=Array(<literal>)`:

```json
{
  "stepType": "Next Item",
  "parameters": {
    "variable": "CurrentPlate",
    "values": "=Array(\"plate1\", \"plate2\", \"plate3\", \"plate4\")",
    "onDone": "Break",
    "setTo": ""
  }
}
```

Cycle and reset to a sentinel value when exhausted:

```json
{
  "stepType": "Next Item",
  "parameters": {
    "variable": "Tube",
    "values": "=TubeList",
    "onDone": "SetTo",
    "setTo": "=\"DONE\""
  }
}
```

Round-robin forever (wrap to the first value — two values minimum):

```json
{
  "stepType": "Next Item",
  "parameters": {
    "variable": "Reagent",
    "values": "=Array(\"A\", \"B\")",
    "onDone": "StartOver",
    "setTo": ""
  }
}
```

Typical loop wiring — the `Loop` supplies the iteration, Next Item supplies the values and
ends the loop when they run out:

```json
{
  "stepType": "Loop",
  "parameters": { "variable": "", "start": "1", "end": "999", "increment": "1" },
  "subSteps": [
    {
      "stepType": "Next Item",
      "parameters": {
        "variable": "CurrentPlate",
        "values": "\"plate1\",\"plate2\",\"plate3\",\"plate4\"",
        "onDone": "Break",
        "setTo": ""
      }
    },
    { "stepType": "Transfer", "parameters": {} },
    { "stepType": "End", "parameters": {} }
  ]
}
```

## Common Mistakes

- **Assuming list elements must be quoted strings**: they need not be — `"1,2,13,14"` is a valid `values` list and numbers are converted to strings. But matching is string-based, so `1` and `01` are different values, and a global holding `1` will not match a list entry of `01`.
- **Relying on the "appears twice" error to catch duplicates**: it only fires when the *current* value matches two entries. Duplicates the sequence never lands on go undetected — keep the list unique yourself.
- **`onDone = "StartOver"` with a one-value list**: fails at enqueue even though the list is not empty. Use two or more values.
- **Misspelling `onDone`**: matching is case-sensitive and unrecognized values silently behave as `"StartOver"` — `"break"` will not break.
- **Expecting `Break` outside a `Loop`**: `onDone = "Break"` calls the break primitive; with no enclosing `Loop` it is uncaught and surfaces as a run-time break error. A `Worklist` does not consume the break — see [Break](49-Break.md).
- **Re-entering a loop without resetting the global**: after a `"Break"` the global still holds the last list value, so the next entry breaks immediately.
- **Confusing `name` with `variable`**: `name` is only a caption alias; `variable` is the global that gets advanced.
