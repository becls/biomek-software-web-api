# Run Method

| Property | Value |
|----------|-------|
| stepType | `"Run Method"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | All |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Envelope / structural rules: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Run Method step runs another method from the current project inline, as part of the
current method. At enqueue time it resolves `filename` against the project's method
collection, rejects a `filename` that matches the full project path of the method currently
open in the editor, evaluates the `let` / `weak` / `prompt` dictionaries into a fresh
environment, attaches that environment to this step's own call node — which
becomes the enclosing scope for the called method's steps — and enqueues those steps into
it. The `let` entries are the argument values handed to the called method.
Run Method is a **leaf** in the JSON tree — no `subSteps`.

**Parameter-passing contract.** A `let` value reaches the called method only if that method's
Start step marks the same variable overridable — that is, lists the same name in its own Start
`weak` dictionary. If it does not, the called method's Start step rebinds the name to its own
`let` value and the caller's value is silently discarded; nothing is reported and the method
simply runs with the wrong value. The names must match.

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `filename` | string (expression-capable) | — | Yes | `""` | Project method name. For a method inside a folder, use the backslash-delimited project path — `Subfolder\MethodName` — which in JSON must be written with an escaped backslash: `"Subfolder\\MethodName"`. Matching is case-insensitive. Do **not** include a drive letter, directory, or a `.bmf` extension (the on-disk Biomek method file): the method is looked up in the current project, not on disk. Omitting the key raises a missing-key SYSTEM ERROR at run time, and an empty string fails to resolve as a method-not-found error. |
| `let` | object | — | Yes | `{}` | Variable bindings passed into the called method. Keys are variable names. Author values as JSON strings: a number as a quoted string (`"4"`), an expression as an `=`-prefixed string (`"=SampleVolume * 2"`). A bare JSON number is accepted on import, but the editor normalizes it back to a string, so string form preserves round-trip fidelity. Omitting the key raises a missing-key SYSTEM ERROR. See §4 of [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md#4-keys-that-look-shared-but-are-genuinely-per-step). |
| `weak` | object | — | Yes | `{}` | Marks `let` entries as weak. Keys are variable names that must also appear in `let`; a key present only here has no effect. Values are flags — author `null`. A weak entry is skipped when the variable is already bound in the calling scope, so the caller's existing value wins and the `let` value acts as a default. Omitting the key entirely raises a missing-key SYSTEM ERROR. |
| `prompt` | object | — | Yes | `{}` | Marks `let` entries that the operator is asked to confirm at run time. Keys are variable names that must also appear in `let`; a key present only here produces no prompt and no variable. Values are flags — author `null`. At run time the operator sees a dialog titled *Enter Value* reading `Enter a value to use for '<name>'`, pre-filled with the `let` value; the response is evaluated and, when it converts to a number, stored as a number. The prompt text is not configurable. Omitting the key entirely raises a missing-key SYSTEM ERROR. |

Universal base keys — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).

This step does **not** carry a `dynamic?` key (unlike Run Procedure). No other step-specific keys are expected on Run
Method — emit only `filename`, `let`, `weak`, and `prompt`.

## Enumerated / Constrained Values

None. `filename` is a free-form project method name; `let`, `weak`, and `prompt` accept any
user-defined variable names as keys.

## Cross-Field Validation Rules

1. **Self-reference is rejected.** If `filename` (case-insensitively) equals the full project path of the currently open method, enqueue fails as a self-reference error. This is a single-level check against the method open in the editor — it is not a general recursion detector, and mutual recursion between two other methods is not caught.
2. **Method must exist.** If the name does not resolve in the project method collection, enqueue fails as a method-not-found error. An empty `filename` fails here like any other unresolvable name.

## Structural Context

Run Method is a **leaf** — it never has child steps of its own. Do **not** include a `subSteps` array
with elements (an omitted array or `[]` is fine). The step does extend the variable
environment, but the environment it extends is the scope the *called* method's steps
run in, not a scope for children of this node.

## Canonical Example

A method stored at the root of the project, with two argument values:

```json
{
  "stepType": "Run Method",
  "parameters": {
    "filename": "Reagent Prep",
    "let": {
      "PlateCount": "4",
      "Volume": "=SampleVolume * 2"
    },
    "weak": {},
    "prompt": {}
  }
}
```

### Foldered method, one weak entry and one prompted entry

`Assays\Serial Dilution` lives in the project folder `Assays`; the backslash is escaped for
JSON. `PlateCount` yields to a same-named variable already bound in the caller's scope, and the
operator is asked to confirm `Volume` (pre-filled with `50`) when the step is enqueued:

```json
{
  "stepType": "Run Method",
  "parameters": {
    "filename": "Assays\\Serial Dilution",
    "let": {
      "PlateCount": "4",
      "Volume": "50"
    },
    "weak": {
      "PlateCount": null
    },
    "prompt": {
      "Volume": null
    }
  }
}
```

## Common Mistakes

- **Treating `filename` as a file path:** it is a project method name, not a location on disk.
  `"C:\\Methods\\Reagent Prep.bmf"` fails as a method-not-found error. Use
  `"Reagent Prep"`, or `"Subfolder\\Reagent Prep"` for a foldered method.
- **Omitting `filename`:** it is required — a missing key raises a SYSTEM ERROR at run time, and
  an empty string fails to resolve.
- **Omitting `let` / `weak` / `prompt`:** all three are required at run time — a missing key
  raises a SYSTEM ERROR. Write `{}` even when empty.
- **Putting prompt text in `prompt`, or values in `weak`:** both are flag sets over `let` keys.
  Their values are never read and their keys do nothing unless the same name is in `let`.
- **Assuming a `let` value always wins:** it is discarded unless the called method's Start step
  marks that variable overridable in its own `weak` dictionary.
- **Confusing with Run Procedure:** Run Method runs a method stored in the current project;
  Run Procedure inlines a procedure defined earlier in the same method by Define Procedure.
