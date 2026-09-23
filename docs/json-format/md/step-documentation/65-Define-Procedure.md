# Define Procedure

| Property | Value |
|----------|-------|
| stepType | `"Define Procedure"` |
| Category | Free-form container |
| Terminator | `"End"` (caption: "End Procedure") |
| Compatible Hardware | All |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Envelope / structural rules: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Define Procedure step declares a **named, reusable procedure** made up of its own child
steps. It does not execute its children where it sits. At enqueue it evaluates and validates
the procedure name, captures its child steps and its `let` bindings as the procedure
definition, and binds that definition to a global variable with that name. A later
`Run Procedure` step resolves that name and inlines the captured body, applying its own `let`
values over the procedure's `let` defaults. Only `let` crosses
that boundary — see the `weak` and `prompt` rows below. Define Procedure is a
**free-form container** whose body (the child steps) is the procedure definition.

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `procedure` | string (identifier, expression-capable) | — | Yes | `""` | Name of the procedure being defined. The value is expression-evaluated first, then checked as an identifier; it must be non-empty and a valid identifier. This name becomes the global variable other steps reference. |
| `let` | object | — | **Yes (include, even as `{}`)** | `{}` | Default (strong) argument bindings for the procedure: keys are variable names, values are literals or `=`-prefixed expressions. These are the values every `Run Procedure` call starts from before applying its own `let` overrides. Author it explicitly — JSON import replaces the step's parameters wholesale, so the step's own default does not fill it in. See also §4 of [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md#4-keys-that-look-shared-but-are-genuinely-per-step). |
| `weak` | object | — | No (include as `{}` to match exports) | `{}` | Weak (fill-in) bindings: name → value used only when that name is not already bound in an enclosing scope. **Persisted for round-trip fidelity, but not applied when the procedure is invoked** — `Run Procedure` builds a fresh scope from the procedure's `let` plus the call's own `let` only, so `weak` authored here has no runtime effect. Put weak defaults on a `Let` step inside the procedure body instead. |
| `prompt` | object | — | No (include as `{}` to match exports) | `{}` | Names to prompt the operator for. Only the key matters: author the value as `null`, because the dialog text is generated from the variable name (`Enter a value to use for '<name>'`) and the stored value is never displayed; a prompted name must also appear in `let`, which supplies the dialog's default. **Not applied when the procedure is invoked** — a prompt declared here produces no dialog; author prompts on a `Let` step inside the procedure body. |

Always author all four keys on a Define Procedure step.

Universal base keys — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
A Define Procedure step may also carry `isPreconfigured` and `caption`;
neither affects procedure configuration.
Define Procedure does not carry a `dynamic?` key — do not author one. (Contrast `Run Procedure`,
which carries `dynamic?: true`.)

## Enumerated / Constrained Values

None enumerated. `procedure` must be a valid identifier: it must start with an ASCII letter
and contain only ASCII letters, digits, and underscores, up to 255 characters. Spaces,
leading digits, leading underscores, and punctuation are rejected.

## Cross-Field Validation Rules

1. **Procedure name required and must be a valid identifier.** If the evaluated `procedure`
   value is empty or is not a valid identifier, enqueue raises an OLE error naming the offending value as an invalid procedure name.
2. **Name must not be shadowed by a `let` variable.** If a `let`/`weak` binding with the same
   name is in scope where the Define Procedure runs, enqueue fails as a shadowing error.
   Redefining the same procedure name is allowed; the later definition replaces the earlier one
   when execution reaches it.
3. **Placement must precede every caller.** The name is bound when the Define Procedure step is *enqueued*, so it must sit ahead of every `Run Procedure` that calls it. A Define Procedure inside an `If` branch or a loop body binds the name only if and when that body runs; otherwise the call fails as a procedure-not-found error.

## Structural Context

Define Procedure is a **free-form container**.

**Constraints** (per [Method JSON Structure](../Method-JSON-Structure.md)):

- `subSteps` must be present and must end with the `"End"` terminator as the last element.
- The steps between the start of the container and the terminator constitute the procedure body.
- The terminator's `stepType` is `"End"`. The terminator's caption should be
  `"End Procedure"`.

## Canonical Example

```json
{
  "stepType": "Define Procedure",
  "parameters": {
    "procedure": "AddReagent",
    "let": {
      "Volume": "100"
    },
    "weak": {},
    "prompt": {}
  },
  "subSteps": [
    { "stepType": "Transfer", "parameters": {} },
    { "stepType": "End", "parameters": { "caption": "End Procedure" } }
  ]
}
```

## Common Mistakes

- **Expecting inline execution:** Define Procedure does not run its body — it registers it in a global for `Run Procedure` to invoke later. A method with only a Define Procedure and no matching Run Procedure does nothing at runtime.
- **Redefining a name:** two Define Procedures with the same `procedure` follow global-set semantics — the later one supersedes when execution reaches it, so which definition a given call sees depends on enqueue order.
- **Reusing a `let` variable's name:** if the procedure name is already bound as a `let`/`weak` variable in scope, enqueue fails as a shadowing error. Pick a name that no enclosing scope binds.
- **Omitting `let`:** this step imports and enqueues without it, but every `Run Procedure` that invokes the procedure then raises a missing-key SYSTEM ERROR. Always author `let`, even as `{}`.
- **Expecting `weak`/`prompt` here to do something:** they round-trip but are not applied when the procedure is invoked — only `let` is. Put weak defaults and operator prompts on a `Let` step inside the procedure body.
- **Non-identifier name via expression:** `procedure` is expression-evaluated and only then validated, so an expression producing `"add reagent"` throws just like the literal would.
