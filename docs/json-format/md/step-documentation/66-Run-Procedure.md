# Run Procedure

| Property | Value |
|----------|-------|
| stepType | `"Run Procedure"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | All |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Envelope / structural rules: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Run Procedure step invokes a procedure previously declared by a `Define Procedure` step.
At enqueue time it evaluates `procedure` to a name, looks it up as a bound global variable
(the Let step stored by Define Procedure), then builds a fresh
Let step, seeds it with the procedure's own default `let` bindings, overrides those with this
step's `let` bindings, copies in the procedure's substeps, and enqueues the result. It is a **leaf** in the JSON tree: it owns no child steps and
inlines the referenced procedure's body at runtime.

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `procedure` | string (identifier, expression-capable) | — | Yes | `""` | Name of the procedure to invoke. |
| `let` | object | — | **Yes (include, even as `{}`)** | `{}` | Override argument bindings for this call — **include it even when empty (`{}`)**. Applied *after* the procedure's defaults, so these win. Keys are variable names; values are literals or `=`-prefixed expressions, and a string constant inside an expression must itself be quoted (`"=\"Empty\""`). See the `let` row of [Let](56-Let.md) for the value-encoding rules. Omitting `let` fails when the step enqueues — see Common Mistakes. |
| `weak` | object | — | No | `{}` | Weak bindings. Persisted and round-tripped, but not consumed by this step — see the note below. A non-empty `weak` is legal and is preserved verbatim, but it is still never read. |
| `prompt` | object | — | No | `{}` | Prompt definitions. Persisted and round-tripped, but not consumed by this step — see the note below. |
| `runProcedure` | boolean | — | No | `true` | **Inactive — not read at runtime (always `true`).** It is never read as configuration — it exists only to distinguish this step from Define Procedure, which the `stepType` already does. Do not rely on it; emit `true` to match exports. |

Every editor-produced Run Procedure step carries all five keys above; author them the same way.

Universal base keys — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
A Run Procedure step may also carry `isPreconfigured` or `caption`, and — where a
generic HTML editor panel has been attached — the `html`/`htmL_*` panel family and `stepUI`. All of
these are inert editor state, not Run Procedure
configuration.

`dynamic?` is a framework-set base key (§2 of the base-keys doc). Every editor-produced Run Procedure step carries `dynamic?: true`. Emit it as `true`, exactly as the canonical example does; do not
compute a value for it.

> **Note on `weak`/`prompt`:** although the shared convention (§4 of the base-keys doc) lists
> `let`/`weak`/`prompt` as genuine per-step keys on procedure steps, this step's enqueue transfers only the `let`
> dictionary into the generated Let step. `weak` and `prompt` are created and round-tripped
> but never read here. Author override values in `let`. (Define
> Procedure, by contrast, deep-copies all three into the procedure definition.)

## Enumerated / Constrained Values

None enumerated. `procedure` must name a variable bound to a procedure defined by an earlier
Define Procedure step.

## Cross-Field Validation Rules

Both errors below are raised while the step enqueues, and both go through the same
enqueue-failure prompt: depending on the answer, the run either stops with the error or
defers this call and continues with the rest of the method.

1. **Procedure must be bound.** If the name is not bound in the current environment when the
   step enqueues, it fails as a procedure-not-found error.
2. **Bound value must be a procedure.** If the name is bound to something that is not a
   procedure (for example a number or string set by Set Global), it fails as a not-a-procedure error.

## Structural Context

Run Procedure is a **leaf** and has no
populated children. Do **not** include a `subSteps` array with elements; the executed steps
come from the referenced Define Procedure's body, inlined at enqueue.

## Canonical Example

```json
{
  "stepType": "Run Procedure",
  "parameters": {
    "procedure": "AddReagent",
    "let": {
      "Volume": "200",
      "SourcePlate": "=\"P1\""
    },
    "weak": {},
    "prompt": {},
    "runProcedure": true,
    "dynamic?": true
  }
}
```

`"200"` is a literal; `"=\"P1\""` is an expression whose result is the string `P1` — a string
constant used inside an `=` expression must carry its own quotes.

## Common Mistakes

- **Calling before defining:** the procedure global must be bound before this step runs
  (Define Procedure earlier in enqueue order), else the step fails as a procedure-not-found error.
- **Putting overrides in `weak`/`prompt`:** only `let` overrides are applied by this step;
  use `let`.
- **Omitting `let` entirely:** enqueue fails as a missing-key error. Include `let` even if empty.
- **Unquoted string constants in an `=` expression:** `"=P1"` reads the *variable* `P1` and
  fails if nothing binds it; write `"=\"P1\""` for the literal text, or drop the `=` entirely.
