# Fly-By Read

| Property | Value |
|----------|-------|
| stepType | `"Fly-By Read"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7 (Fly-By Bar Code Reader is an i5/i7 device; **not** i3) |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Envelope / structural rules: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Fly-By Read step uses a gripper-equipped pod to pick up a stack of labware
from a `source` deck position, "fly" each piece past a stationary Fly-By Bar
Code Reader (FBBCR) device to read its bar code into the labware's
`Properties.Barcode`, and then set the stack down at a `target` deck position.
Common uses are entering bar codes for later decision-making (e.g. a downstream
`If` step) and confirmatory reads that the correct labware was selected. Reads
are accumulated for later use by the Fly-By Log step.

A Fly-By Bar Code Reader must be installed on the deck to use this step.

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | string (expression) | — | Yes | `""` | Name of the gripper-equipped pod used to move the stack (e.g. `"Pod1"`). Expression-evaluated at enqueue. |
| `source` | string (expression) | — | Yes | `""` | Deck position from which to pick up the labware stack. Expression-evaluated. |
| `target` | string (expression) | — | Yes | `""` | Deck position at which to place the stack after reading. Expression-evaluated. |
| `device` | string (expression) | — | Yes | `""` | Name of the Fly-By Bar Code Reader device. Expression-evaluated. Must be a Fly-By Bar Code Reader configured on the current deck. |
| `gripSide` | string (enum) | — | Yes | `""` | Gripper orientation used to grip the stack: `"A1 near"` or `"A1 away from"`. Matched case-insensitively. See Enumerated Values. |
| `simulatedBarcode` | string (expression) | — | Yes | `"Simulating"` (editor pre-fill; enqueue fallback is `"Simulate"` when the key is absent) | Bar code value returned during simulation / method validation. Expression-evaluated. |
| `returnLid` | boolean | — | Yes | `true` | If `true`, the lid removed from a lidded stack is replaced onto the plate after reading. **Genuine JSON boolean**. |
| `override` | **string** `"True"`/`"False"` | — | Yes | `"False"` (consumed as `False` at enqueue) | Whether to override the device's default error handling. **This is a `"True"`/`"False"` string, not a JSON boolean** — write `"True"`/`"False"`, not `true`/`false`. |
| `errorHandling` | string (enum) | — | Yes | `"Ignore"` | Error-resolution mode used **only when `override` is `"True"`**: `"Ignore"`, `"Barcode"`, or `"Prompt"`. |
| `retries` | **string** (integer, expression) | count | Yes | `"1"` | Number of retries when a read error or mismatch occurs. Persisted as a string; evaluated as an integer at enqueue. **Used only when `override` is `"True"`**; otherwise the device's own `Retries` (default `1`) is used. |
| `overrideBarcode` | string (expression) | — | Yes | `""` | Bar code value substituted when `errorHandling` is `"Barcode"`. Expression-evaluated. |
| `leaveBottomLabware` | boolean | — | Yes | `false` | If `true`, the stack is moved but the bottom piece of labware is left at the `source` position (moves `StackDepth − 1` pieces). Always emitted by the editor. Mutually exclusive with `depth` (see Stack Options). |
| `depth` | **string** (integer) | count | Conditional | *(absent)* | Number of topmost pieces of labware to move from the stack. Present **only** when the "Move depth" option is chosen. When absent (and `leaveBottomLabware` is `false`), the entire stack is moved. Mutually exclusive with `leaveBottomLabware`. |

Universal base keys (`caption`, node `disabled`, etc.) — see
[Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md). `dynamic?` is always
written by this step's editor (so the canonical examples below include it for round-trip
fidelity). Include `dynamic?` with value `true`.

## Enumerated / Constrained Values

### gripSide

| Value | Meaning |
|-------|---------|
| `"A1 near"` | Grip with well A1 nearest the pod. |
| `"A1 away from"` | Grip with well A1 away from the pod. |

The exact source constant values are `'A1 near'` and `'A1 away from'`. Matching is case-insensitive, but these
are the values the editor writes, so author them verbatim. The editor lists `"A1 away from"`
first, so it is the preferred/default choice when both are geometrically valid.

### errorHandling (effective only when `override` is `"True"`)

| Value | Meaning |
|-------|---------|
| `"Ignore"` | Ignore the error and keep the (possibly failed) bar code; continue. |
| `"Barcode"` | Substitute the `overrideBarcode` value for the read. |
| `"Prompt"` | Prompt the user (Retry / Ignore / Abort / enter a bar code); also enables duplicate-bar-code detection between consecutive plates. |

The default (used when the string is unrecognized) is `"Ignore"`. `"Barcode"` is a valid option — author it (with `overrideBarcode`) when substitution is intended.

### Stack Options (how `leaveBottomLabware` + `depth` select what moves)

| Configuration | Pieces moved |
|---------------|--------------|
| `leaveBottomLabware: true` | `StackDepth − 1` (bottom piece stays at `source`). |
| `depth` absent, `leaveBottomLabware: false` | Entire stack (`StackDepth`). |
| `depth: "N"` | Topmost `N` pieces. |

The three options are mutually exclusive.

## Cross-Field Validation Rules

1. `device` must be set.
2. `device` must resolve in `PipettorDevices`.
3. `device` must be associated with a deck position — see [Deck Layouts Format](../biomek-file-formats/deck-layouts/deck-layouts-format-spec.md#the-device-block) §"The `device` block" for how the association is set up outside the method.
4. `gripSide` must be one of the two enum strings.
5. `retries` must be numeric when `override` is `"True"`.
6. `source` must be non-empty.
7. `pod` must resolve.
8. `source` must be a position on the current deck.
9. `target` must be a position on the current deck.
10. `source` must hold labware.
11. `pod` must have a gripper.
12. `device` must be on the current deck.
13. Stack must contain at least one non-lid plate to read.
14. A lidded stack requires a swap position for the removed lid.

## Structural Context

This step is a **leaf** and does not support authored `subSteps`.

## Canonical Example

Reads the whole stack at P2, overriding error handling to prompt after 3 retries, and places
the stack at P3:

```json
{
  "stepType": "Fly-By Read",
  "parameters": {
    "pod": "Pod1",
    "source": "P2",
    "target": "P3",
    "device": "FBBCR1",
    "gripSide": "A1 away from",
    "simulatedBarcode": "PLATE001",
    "returnLid": true,
    "leaveBottomLabware": false,
    "override": "True",
    "errorHandling": "Prompt",
    "retries": "3",
    "overrideBarcode": "",
    "dynamic?": true
  }
}
```

Minimal read using the device's default error handling and moving only the top 2 pieces:

```json
{
  "stepType": "Fly-By Read",
  "parameters": {
    "pod": "Pod1",
    "source": "P2",
    "target": "P2",
    "device": "FBBCR1",
    "gripSide": "A1 near",
    "simulatedBarcode": "TEST",
    "returnLid": false,
    "leaveBottomLabware": false,
    "depth": "2",
    "override": "False",
    "errorHandling": "Ignore",
    "retries": "1",
    "overrideBarcode": "",
    "dynamic?": true
  }
}
```

## Common Mistakes

- **Writing `override` as a JSON boolean.** It is a `"True"`/`"False"` **string** — use
  `"True"`/`"False"`, never `true`/`false`. (`returnLid`, by contrast, *is* a real JSON boolean.)
- **Writing `retries` or `depth` as JSON numbers.** Both are persisted as strings (`"3"`, `"2"`).
- Setting `errorHandling` / `retries` / `overrideBarcode` while leaving `override: "False"` — those
  three keys are consumed only when `override` is `"True"`; otherwise the device's own defaults win.
