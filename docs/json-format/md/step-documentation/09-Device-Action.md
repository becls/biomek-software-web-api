# Device Action

| Property | Value |
|----------|-------|
| stepType | `"Device Action"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7 |

## Behavior Summary

The Device Action step sends a command to a digital I/O device or generic script device attached to the pipettor. It looks up the named device in the pipettor's device collection, then dispatches the command string. For digital I/O devices it dispatches the command without parameters. For generic script devices it dispatches the command with a required parameter array (`parameters` key must be present in the step node). This is the general-purpose step for controlling non-SILAS peripheral devices (e.g., control boxes, orbital shakers, custom hardware).

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `device` | `string` | — | Yes | `""` | Device name. Must exist in the pipettor's device collection. See [Deck Layouts Format](../biomek-file-formats/deck-layouts/deck-layouts-format-spec.md#the-device-block) §"The `device` block" for where to find the valid names on your instrument. |
| `command` | `string` | — | Yes | `""` | Command to send to the device. |
| `parameters` | `array of strings` | — | Conditional (when device is a generic script device) | *(not bound)* | Parameter array passed to the device command. Omission raises a "could not find Parameters" system error when that branch runs. Not read for digital I/O devices. |
| `slack` | numeric | seconds | No | Not present | **Inert on this step.** `slack` on a Device Action node is stray persistence with no runtime effect — safe to omit. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

## Enumerated / Constrained Values

The Device Action step has **no enumerated or string-constrained parameters**. `device`, `command`, and (for generic script devices) `parameters` are free-form strings whose accepted values are entirely defined by the target device driver, not by this step. Consult the specific device's documentation for the command vocabulary and any parameter-array conventions it expects.

The two supported device categories are dispatched internally and are not authorable values:

| Device category | Behavior |
|-----------------|----------|
| Digital I/O device | Command dispatched; `parameters` key is ignored. |
| Generic script device | Command dispatched with the parameter array; `parameters` key is required. |

## Cross-Field Validation Rules

1. `device` **must not** be empty.
2. `device` **must** exist in the pipettor's device collection.
3. `command` **must not** be empty.
4. The device must be either a digital I/O device or a generic script device. If it is neither, enqueue fails with a "device not supported" error.

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

Simple device command:

```json
{
  "stepType": "Device Action",
  "parameters": {
    "device": "ControlBox",
    "command": "TurnOn"
  }
}
```

With parameters (generic script device):

```json
{
  "stepType": "Device Action",
  "parameters": {
    "device": "OrbitalShaker",
    "command": "SetSpeed",
    "parameters": ["500", "60"]
  }
}
```

> **Note:** The `parameters` key here is a step-specific key that sits *inside* the universal `parameters` envelope object — the same name appears at two different nesting levels in the JSON.

## Common Mistakes

- **Using on i3**: This step is compatible only with i5 and i7.
- **Omitting `parameters` for a generic script device**: For generic script device targets the key is read unconditionally in that enqueue branch and throws when missing; conversely, for digital I/O device targets the key is never read (so leaving it in is harmless but stray).
- **Authoring `slack` on this step**: see §Parameters row `slack`. Omit it; the key is harmless but has no runtime effect on this step.
