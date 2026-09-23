# Set Global

| Property | Value |
|----------|-------|
| stepType | `"Set Global"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | All |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Structure/envelope: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Set Global step defines or updates a **global** variable. Unlike [Let](56-Let.md), whose bindings are visible only inside that container, a global is not tied to any container's scope. At **enqueue** (validate) the step evaluates `name` to a variable name and `value` to a value, then stores the pair in the run's global variable table. The binding is established during the enqueue pass, so any step enqueued after this one — at any nesting depth — can reference it, and re-running Set Global with the same `name` simply overwrites the value.

Globals outlive the run: they are cleared by a [Finish](02-Finish.md) step with `clearGlobals` set to `true` (its default), which is what prevents carry-over into the next method.

If `name` is already bound by a **scope** binding in effect at this point — a Start step variable, a `let` / `weak` / `prompt` binding on an enclosing container, a Loop counter, or a Run Method / Run Procedure argument — the step raises the shadowing error (Cross-Field rule 2). An existing **global** of the same name is not shadowing; it is simply updated.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `name` | `expression-capable string` | — | Yes | *(none — must be authored)* | The global variable name. Passed through the expression evaluator: a bare string is used verbatim (leading/trailing whitespace trimmed) as the name; a `=`-prefixed string is **evaluated** and its *result* becomes the name (indirection — see Common Mistakes). Must be a valid identifier — see Cross-Field rule 3. |
| `value` | `expression-capable string` | — | No | `""` | The value to assign. A bare string is stored verbatim (whitespace-trimmed) as a **string**; a `=`-prefixed string is evaluated and the **result type** is stored. So `"25"` stores the string `"25"` while `"=25"` stores the number `25` — use the `=` form when later steps do arithmetic or numeric comparison on the global. The engine tolerates omission — an omitted `value` behaves as `""`. |

## Enumerated / Constrained Values

No enumerated constraints on `value`. `name` is constrained to identifier syntax (Cross-Field rule 3). Both keys accept literal strings or `=`-prefixed expressions.

## Cross-Field Validation Rules

1. `name` **must** be present in `parameters` — otherwise the step fails as a not-configured error.
2. `name` **must not** be shadowed by a scope binding in effect at this point — otherwise the step fails as a shadowing error. The check covers **any** name currently bound in the variable environment: a Start step variable, a `let` / `weak` / `prompt` binding, a Loop counter, or a Run Method / Run Procedure argument. An already-existing global of the same name does **not** trigger it.
3. `name` **must** evaluate to a valid identifier — it begins with a letter, then letters, digits, or underscores only, 1 to 255 characters. Otherwise the step fails as an invalid-variable-name error. An empty `name` fails this same check.

## Structural Context

This step is a **leaf** and does not support `subSteps`. Include no `subSteps` (or an empty array).

## Canonical Examples

Set a numeric global (the `=` prefix makes the stored value the **number** 4, not the string `"4"`):

```json
{
  "stepType": "Set Global",
  "parameters": {
    "name": "PlateCount",
    "value": "=4"
  }
}
```

Set a literal text global (stored as the string `"Plate A"`):

```json
{
  "stepType": "Set Global",
  "parameters": {
    "name": "CurrentPlateLabel",
    "value": "Plate A"
  }
}
```

Set a global using an expression:

```json
{
  "stepType": "Set Global",
  "parameters": {
    "name": "RemainingVolume",
    "value": "=TotalVolume - UsedVolume"
  }
}
```

Clear a global's value. The variable stays defined and holds an empty value — it is not un-defined. [Next Item](60-Next-Item.md) treats an empty value as "start from the first item":

```json
{
  "stepType": "Set Global",
  "parameters": {
    "name": "Status",
    "value": ""
  }
}
```

## Common Mistakes

- **String-typed numbers**: `"value": "4"` stores the string `"4"`, not the number 4. Prefix numeric values with `=` (`"=4"`) when downstream steps do arithmetic or numeric comparison on the global. Both spellings occur in practice, and some methods work around the string form with an explicit conversion such as `"=cDbl(SomeGlobal)"`.
- **Literal-vs-expression `value`**: an unprefixed value is a literal string; prefix with `=` to evaluate (`"=Count + 1"`, not `"Count + 1"`).
- **Prefixing `name` with `=`**: `"name": "=PlateCount"` sets a global whose *name* is the current value of `PlateCount` (indirection), not the global `PlateCount`. Author the name bare: `"name": "PlateCount"`.
- **Invalid name characters**: no spaces, hyphens, dots, or leading digits in `name` — the step hard-fails at enqueue (Cross-Field rule 3).
- **Shadowed variable**: if any binding in scope already uses the name — a [Start](01-Start.md) step variable, a `let`/`weak`/`prompt` binding, a Loop counter, or a Run Method / Run Procedure argument — Set Global fails with the shadowing error (Cross-Field rule 2). Reusing a Start-step variable name is the most common way to hit this. Rename the global, or drop the Start variable and set it here instead.
- **Colliding with a procedure name**: [Define Procedure](65-Define-Procedure.md) registers its procedure as a global of the same name. Setting a global with a procedure's name overwrites it, and the matching [Run Procedure](66-Run-Procedure.md) then fails as a not-a-procedure error. Keep global names and procedure names distinct.
- **Confusing with Let**: [Let](56-Let.md) creates scoped variables for child steps. Set Global creates run-wide globals. Use Set Global when you need a variable accessible outside the current container.
