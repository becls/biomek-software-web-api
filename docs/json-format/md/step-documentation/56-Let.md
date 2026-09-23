# Let

| Property | Value |
|----------|-------|
| stepType | `"Let"` |
| Category | Container (free-form) |
| Terminator | `"End"` (caption: "End Let") |
| Compatible Hardware | All |

## Behavior Summary

The Let step binds scoped variables for its child steps. At run time it evaluates each entry in `let`, applies the `weak` and `prompt` flags for any matching names, extends the environment with the resulting bindings, and enqueues its children in that extended environment. A `let` entry is a **strong** binding — it shadows any outer definition of the same name. Listing that name in `weak` makes the binding defer to an existing outer definition; listing it in `prompt` asks the operator for the value at run time, using the `let` value as the dialog default. Only `let` holds values: `weak` and `prompt` are per-name flag sets whose keys must also appear in `let`. When execution leaves the Let container, the bindings go out of scope.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `let` | `object` | — | **Yes** | `{}` | Dictionary of variable definitions — the only one of the three keys that holds values. Each property name becomes a scoped variable; each property value is a literal or an `=`-prefixed expression, evaluated at run time. Author values as JSON strings (e.g. `"50"`). JSON numbers and booleans are accepted, but only a **string** value goes through the expression evaluator — a non-string is passed through unchanged, so an `=`-expression only works when the value is a string. **Must be present** (empty `{}` is fine); omitting it fails at Run on the missing-key lookup. |
| `weak` | `object` | — | **Yes** | `{}` | Dictionary of **weak-binding flags** — not an independent variable map. A variable listed here binds only if the name is **not** already defined in an enclosing scope; use it for defaults a caller can override. Keys are variable names that **must also appear in `let`** (the value lives in `let`); values should be `null` — presence of the key is the flag, and the value stored here is never read. A `weak` key with no matching `let` key has no effect. **Must be present** (empty `{}` is fine); omitting it fails at Run on the missing-key lookup. |
| `prompt` | `object` | — | **Yes** | `{}` | Dictionary of **runtime-prompt flags**. A variable listed here raises a dialog at run time letting the operator confirm or change its value. Keys are variable names that **must also appear in `let`** (the `let` value supplies the dialog default); values should be `null` — presence of the key is the flag. The dialog message and title are generated automatically; the value you store here is not used. A `prompt` key with no matching `let` key produces no dialog. The operator's answer is evaluated exactly like a `let` value (a leading `=` makes it an expression) and is then converted to a number whenever it is numeric-looking — so an answer meant as text but written as digits (a leading-zero barcode, for instance) comes back as a number. A name flagged in both `weak` and `prompt` is **not** prompted when an outer scope already defines it — the weak check runs first. **Must be present** (empty `{}` is fine); omitting it fails at Run on the missing-key lookup. |

Each of the three keys is a plain JSON object: the property name is the variable name; the property value is the value or `=`-expression (for `let`) or `null` (for `weak` / `prompt`). Every editor-produced Let step carries all three keys; author new JSON the same way.

> _Universal base keys (`caption`, `isPreconfigured`, `dynamic?`, `stepUI`, `disabled`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

See also [Concept Guide: The Variable Environment (`let` / `weak` / `prompt`)](../concept-guides/04-variable-environment.md) for how variable scopes nest across all scope steps.

## Enumerated / Constrained Values

No enumerated value constraints. `let` values are literals or `=`-prefixed expressions; `weak` and `prompt` values are unused flags and should be `null`.

## Cross-Field Validation Rules

1. `let` **must** contain at least one entry for the step to have any effect — entries in `weak` or `prompt` alone do nothing. An empty Let (all three `{}`) is structurally valid and simply executes its children unchanged.
2. Every key in `weak` **must** have a corresponding key in `let`. A `weak` key with no `let` entry never binds.
3. Every key in `prompt` **must** have a corresponding key in `let`. A `prompt` key with no `let` entry produces no dialog.
4. Variable names **should** be valid script identifiers — no spaces, no leading digits *(not enforced at enqueue but will cause failures in child steps that reference invalid names)*.
5. Expression values (prefixed with `=`) **must** be syntactically valid
   VBScript/JScript expressions. Use `=` for VBScript and `==` for JScript.

## Structural Context

The Let step is a **free-form container**. It requires one child:

- **Trailing anchor**: an `"End"` step (caption `"End Let"`) is required as the container's last child. Author it as the last element of `subSteps`.

**Constraints**:

- `subSteps` must be present and contain at least the `"End"` terminator as the last element.
- Any number of operational steps may be placed before the `"End"` terminator.
- Child steps reference the variables defined in `let` via `=VariableName` syntax.

## Canonical Examples

Simple Let defining two variables:

```json
{
  "stepType": "Let",
  "parameters": {
    "let": {
      "Volume": "100",
      "TipType": "P200"
    },
    "weak": {},
    "prompt": {}
  },
  "subSteps": [
    { "stepType": "Transfer", "parameters": {} },
    { "stepType": "End", "parameters": { "caption": "End Let" } }
  ]
}
```

Let with expression values:

```json
{
  "stepType": "Let",
  "parameters": {
    "let": {
      "HalfVolume": "=TotalVolume / 2",
      "DestColumn": "=SourceColumn + 1"
    },
    "weak": {},
    "prompt": {}
  },
  "subSteps": [
    { "stepType": "Span-8 Aspirate", "parameters": {} },
    { "stepType": "Span-8 Dispense", "parameters": {} },
    { "stepType": "End", "parameters": { "caption": "End Let" } }
  ]
}
```

Let with weak bindings (defaults that an outer scope can override). The values live in `let`; `weak` carries only the flags:

```json
{
  "stepType": "Let",
  "parameters": {
    "let": {
      "DefaultVolume": "50",
      "MixCycles": "3"
    },
    "weak": {
      "DefaultVolume": null,
      "MixCycles": null
    },
    "prompt": {}
  },
  "subSteps": [
    { "stepType": "Transfer", "parameters": {} },
    { "stepType": "End", "parameters": { "caption": "End Let" } }
  ]
}
```

Let that prompts the operator. `SampleCount` is defined in `let` (that value is the dialog default) and flagged in `prompt`:

```json
{
  "stepType": "Let",
  "parameters": {
    "let": {
      "SampleCount": "96"
    },
    "weak": {},
    "prompt": {
      "SampleCount": null
    }
  },
  "subSteps": [
    { "stepType": "Transfer", "parameters": {} },
    { "stepType": "End", "parameters": { "caption": "End Let" } }
  ]
}
```

## Common Mistakes

- **Forgetting the `"End"` terminator**: Let is a free-form container and must end with `"End"`.
- **Using `let` as a flat key-value instead of a nested object**: Write `"let": { "Var": "value" }`, not `"let": "Var=value"`.
- **Putting values in `weak` or `prompt`**: only `let` holds values. A name that appears in `weak` or `prompt` but not in `let` never binds and never prompts — the value stored under that key is never read.
- **Confusing `let` with `weak`**: `let` is the strong binding that always takes effect; adding the same name to `weak` makes it defer to an outer-scope definition.
- **Literal vs. expression values**: Bare strings are literals; expressions must be prefixed with `=` (e.g., `"=TotalVolume / 2"`).
