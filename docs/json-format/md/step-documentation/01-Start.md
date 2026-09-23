# Start

| Property | Value |
|----------|-------|
| stepType | `"Start"` |
| Category | Boundary |
| Terminator | N/A |
| Compatible Hardware | All (no hardware restriction — inherits default) |

---

## Behavior Summary

The Start step is the mandatory first step in every Biomek method. It serves two purposes: (1) it defines method-level variables (`let`), weak-bound defaults (`weak`), and runtime prompts (`prompt`) that are available to all subsequent steps in the method; and (2) at runtime, when this Start is confirmed to be the first substep of the method root, it performs hardware initialization — notifying all pipettor devices, reserving the universal resource to ensure safe initialization, resetting piercing-tip state on Span-8 pods, and coercing any out-of-limit pod axes back within their physical bounds.

If the Start step is not the first substep (e.g., in a non-root context), it does nothing at enqueue time.

The Start step also supports SILAS module initialization and method extension enqueuing. Users cannot delete, move, copy, or insert steps above the Start step in the editor.

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `let` | `object` | — | Yes | `{}` (empty dictionary) | Dictionary of method-level variable definitions. Keys are variable names (alphanumeric + underscore, starting with a letter, max 255 chars); values are the variable's default value. Author as a JSON string (e.g., `"25"`); JSON numbers are accepted at runtime but the editor normalizes them to strings, so string form preserves round-trip fidelity. These variables are accessible from any step in the method via `=variableName`. |
| `weak` | `object` | — | Yes | `{}` (empty dictionary) | Dictionary of weak-bound variable flags. A variable listed here is "overridable" — its value in `let` can be replaced by a same-named variable from a calling method (via Run Method step). Keys are variable names matching entries in `let`; values should be `null`. Presence of the key is the flag. |
| `prompt` | `object` | — | Yes | `{}` (empty dictionary) | Dictionary of runtime-prompt variable flags. A variable listed here causes a dialog prompt at method start allowing the operator to confirm or change its value. Keys are variable names matching entries in `let`; values should be `null`. Presence of the key is the flag. |
| `silasModules` | `object` | — | No | Not present | SILAS module and consumer initialization data. Keys are device/consumer identifiers; values are either `true` (for SILAS devices) or serialized SILAS message objects (for SILAS consumers). Treat SILAS consumer payloads as opaque unless specifically editing known SILAS content. This key is the serialized form of an internal extension entry, renamed during export. |
| `extensions` | `object` | — | No | Not present | Container for registered method-extension data, keyed by extension class ID. Each extension receives its sub-dictionary during enqueue. Typically not authored manually. |
| `bitmap` | `string` | — | Yes | `"OStepUI.ocx,START"` | Step-icon reference. Defaults to `"OStepUI.ocx,START"`, but JSON import replaces the step's parameters wholesale and discards defaults — if omitted from authored JSON the Start icon breaks on import. Author it exactly as shown. |

> _Other universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

---

## Enumerated / Constrained Values

The Start step has no parameters with fixed enumerated value sets. The `let`, `weak`, and `prompt` dictionaries accept arbitrary user-defined variable names as sub-keys with arbitrary string/numeric values (for `let`) or `null` flags (for `weak`/`prompt`).

---

## Cross-Field Validation Rules

1. Every key in `weak` **must** have a corresponding key in `let`. A weak flag without a defined variable value has no effect and indicates an authoring error.
2. Every key in `prompt` **must** have a corresponding key in `let`. A prompt flag without a defined variable value will prompt for an undefined variable.
3. The `silasModules` key should only be present if SILAS hardware is configured on the instrument. Including it on a non-SILAS system will not cause a failure but has no effect.
4. The `extensions` key should not be hand-authored — it is populated by the system based on registered method extensions. Including unknown extension CLSIDs will be silently ignored at enqueue.

---

## Structural Context

The Start step is a **Boundary** step with the following structural constraints:

- **Must be at index 0** of the method root `steps` array. The method is always structured as `[Start, ...body steps..., Finish]`.
- **Exactly one Start step** per method. Duplicates are invalid.
- **Root-only**: The Start step must never appear inside a container step's `subSteps`.
- **Cannot be deleted, moved, copied, or have steps inserted above it**.
- **Does not have `subSteps`** — the Start step is not a container. Never include a `subSteps` array on it.

---

## Canonical Examples

### Minimal Start (empty parameters — the most common case)

```json
{
  "stepType": "Start",
  "parameters": {
    "bitmap": "OStepUI.ocx,START",
    "let": {},
    "weak": {},
    "prompt": {}
  }
}
```

### Start with method-level variables and runtime prompts

A serial dilution method where the operator is prompted for volume and plate count at runtime, and the plate count can be overridden by a calling method:

```json
{
  "stepType": "Start",
  "parameters": {
    "bitmap": "OStepUI.ocx,START",
    "let": {
      "aspirateVolume": "25",
      "numberOfPlates": "4",
      "tipType": "BC190F",
      "rows": "\"1,2,3,4,5,6,7,8\""
    },
    "weak": {
      "numberOfPlates": null
    },
    "prompt": {
      "aspirateVolume": null,
      "numberOfPlates": null
    }
  }
}
```

### Start for a sub-method (weak defaults only, no prompts)

A utility method intended to be called via Run Method, where all variables are overridable by the caller:

```json
{
  "stepType": "Start",
  "parameters": {
    "bitmap": "OStepUI.ocx,START",
    "let": {
      "dilutionFactor": "2",
      "mixCycles": "3",
      "sourcePosition": "P1"
    },
    "weak": {
      "dilutionFactor": null,
      "mixCycles": null,
      "sourcePosition": null
    },
    "prompt": {}
  }
}
```

### Start with SILAS module initialization

```json
{
  "stepType": "Start",
  "parameters": {
    "bitmap": "OStepUI.ocx,START",
    "let": {
      "sampleCount": "96"
    },
    "weak": {},
    "prompt": {
      "sampleCount": null
    },
    "silasModules": {
      "orbitShaker1": true,
      "plateReader1": true
    }
  }
}
```

---

## Common Mistakes

- **Omitting `bitmap`** — Import replaces the step's parameters wholesale and discards the default icon value; if `bitmap` is missing the Start icon breaks on the next open. Set it exactly to `"OStepUI.ocx,START"`.
- **Numeric `let` defaults as JSON numbers instead of strings** — Both are accepted at runtime (values are expression-capable), but the editor stores them as strings; using strings preserves round-trip fidelity.
- **Confusing `let` with `weak`** — `let` holds the actual value; `weak` is a parallel flag dictionary (values are `null`) marking which `let` entries a caller may override.
- **Forgetting escaped quotes on multi-value row/column strings** — When a variable feeds a Select Tips row/column list, the value itself must include the double-quotes: `"\"1,2,3\""`. A bare `"1,2,3"` will not parse.
