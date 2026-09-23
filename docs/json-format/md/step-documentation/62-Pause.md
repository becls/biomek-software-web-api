# Pause

| Property | Value |
|----------|-------|
| stepType | `"Pause"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | All |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).

## Behavior Summary

The Pause step halts method execution either until the user acknowledges a prompt or for a timed duration on a specific resource.

In `"PromptedGlobal"` mode the step saves the pod positions and pauses the
interlock/light curtain, displays `message` and waits for the user to acknowledge it, then
resumes the interlock/light curtain and restores the pod positions.

In `"TimedResource"` mode the step pauses a single resource — the whole system, a pod, or a deck position — for `time` seconds, which lets other resources keep working in parallel while that resource waits. No prompt is shown, and pod positions and the interlock/light curtain are not touched.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `mode` | `string` | — | Yes | `"User"` — an internal default the run time rejects; always overwrite it | Pause mode: `"PromptedGlobal"` or `"TimedResource"` — see Mode Values below. **Not expression-evaluated**: the literal value is compared (case-insensitively) against those two names, so `"=someVariable"` is never resolved and always fails. Emit the canonical spelling. |
| `message` | `expression-capable string` | — | Conditional | `"Paused"` *(internal default — see note below)* | Message displayed to the user. Read only when `mode` is `"PromptedGlobal"`, and required then. Evaluated as an expression. |
| `time` | `expression-capable string` | seconds | Conditional | `"0"` | Duration to pause, in seconds. Read only when `mode` is `"TimedResource"`, and required then. Emit it quoted — either a constant (`"30"`) or an expression (`"=IncubationTime"`). |
| `location` | `expression-capable string` | — | Conditional | *(no default)* | The resource to pause. Read only when `mode` is `"TimedResource"`, and required then. Evaluated as an expression, so a literal name is written plain (`"P4"`) and a computed one is written with a leading `=` (`"=\"AMPure plate\""`). See Location Values below. |

> **Emit all four keys.** Every Pause step carries `mode`, `message`, `time`, and `location`: `"PromptedGlobal"` steps carry the unused `location` (`""`), and `"TimedResource"` steps carry the unused `message` (`"Paused"`). These defaults do **not** survive a JSON import — the imported parameter set replaces the step's property dictionary wholesale, so a key you omit is simply absent, and omitting a key the chosen mode reads fails at run time. Emitting all four keys avoids that failure.

## Enumerated / Constrained Values

### mode Values

| Value | Meaning |
|-------|---------|
| `"PromptedGlobal"` | Pause everything and display `message`; wait for user acknowledgment. |
| `"TimedResource"` | Pause a single resource for `time` seconds. |

> These are the only two values the run time accepts. Anything else — including the internal default `"User"` — fails at run time; see Common Mistakes.

### location Values (when mode is "TimedResource")

| Value | Meaning |
|-------|---------|
| `"the whole system"` | Universal pause — pauses every resource |
| `"Pod1"` | Pauses Pod 1 only (on a single-pod instrument, the only pod) |
| `"Pod2"` | Pauses Pod 2 only (dual-pod instruments only) |
| *(deck position name)* | Pauses that deck position |
| *(labware name)* | Resolved to the deck position that currently holds labware of that name; that position is paused |

Notes on `location`:

- The three special names above are matched case-insensitively; everything else is treated as a labware or deck-position name.
- `"Pod1"` and `"Pod2"` are valid only if that pod exists on the configured instrument. Naming a pod that is not installed fails at run time. Single-pod instruments — including the i3 Fixed-8 — address their pod as `"Pod1"`.
- A name that is neither a special name, nor labware currently on the deck, nor a bound deck position fails at run time as a position-not-found error.
- A labware name (e.g. `"=\"AMPureXP plate\""`) resolves to whichever deck position currently holds that labware.

## Cross-Field Validation Rules

1. `mode` **must** be either `"PromptedGlobal"` or `"TimedResource"`. Any other value fails at run time as a not-configured error.
2. When `mode` is `"TimedResource"`, `location` **must** be present and name a resource that exists. An empty `location` fails at enqueue as a location-not-specified error; a name that resolves to no deck position fails as a position-not-found error.
3. When `mode` is `"TimedResource"`, `time` should be set to a positive value *(a time of 0 is technically valid but produces a zero-duration pause)*.
4. When `mode` is `"PromptedGlobal"`, `message` **must** be present. It is read unconditionally in that branch, and the default does not survive JSON import.

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

User pause with a message:

```json
{
  "stepType": "Pause",
  "parameters": {
    "mode": "PromptedGlobal",
    "message": "Please add reagent to the reservoir and click OK to continue.",
    "time": "0",
    "location": ""
  }
}
```

Timed pause on the whole system for 30 seconds:

```json
{
  "stepType": "Pause",
  "parameters": {
    "mode": "TimedResource",
    "message": "Paused",
    "time": "30",
    "location": "the whole system"
  }
}
```

Timed pause on a specific deck position (incubation):

```json
{
  "stepType": "Pause",
  "parameters": {
    "mode": "TimedResource",
    "message": "Paused",
    "time": "300",
    "location": "P4"
  }
}
```

Timed pause on the position currently holding a named plate, using an expression for duration:

```json
{
  "stepType": "Pause",
  "parameters": {
    "mode": "TimedResource",
    "message": "Paused",
    "time": "=IncubationTime",
    "location": "=\"AMPureXP plate\""
  }
}
```

## Common Mistakes

- **Emitting `"User"` as `mode`** — the run time recognizes only `"PromptedGlobal"` and `"TimedResource"` and fails as not-configured on anything else. Always overwrite the internal default.
- **Treating `time` as milliseconds** — `time` is in **seconds**. A five-minute incubation is `"300"`, not `"300000"`.
- **Authoring `time` as a JSON number** — the editor always writes `time` as a string, whether it is a constant (`"30"`) or an expression (`"=IncubationTime"`). Quote it in both cases to preserve round-trip fidelity.
- **Putting an expression in `mode`** — `mode` is the one key on this step that is not expression-evaluated. `"=modeVar"` is compared literally against the two accepted names, does not match, and fails as "not properly configured".
- **Omitting the keys the chosen mode does not read** — the imported parameter set replaces the step's dictionary outright, so omitted keys are absent rather than defaulted. Emit all four.
