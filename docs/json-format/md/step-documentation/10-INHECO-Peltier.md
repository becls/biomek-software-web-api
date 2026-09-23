# INHECO Peltier

| Property | Value |
|----------|-------|
| stepType | `"INHECO Peltier"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7 |

## Behavior Summary

The INHECO Peltier step controls INHECO thermal and shaking devices on the Biomek deck. It sends commands to a specific deck position's associated INHECO device for temperature incubation, shaking, or initialization.

Devices can be static peltiers (temperature only) or shaking peltiers (temperature + orbital shaking). SingleTEC devices have different parameter ranges than older models.

## Parameters Reference Table

> _Several keys on this step carry **literal internal spaces** — `use Temp`, `temp Set Point`, `shake RPM`, `continue Shaking`, `deluxe Shake` — and must be written exactly that way, spaces included. The reflexive camelCase forms (`useTemp`, `tempSetPoint`, …) are not recognized. See the key-name casing rule in [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

### Core Parameters

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `position` | `expression-capable string` | — | Yes | `""` | Deck position name. Must have an associated INHECO device — see [Deck Layouts Format](../biomek-file-formats/deck-layouts/deck-layouts-format-spec.md#the-device-block) §"The `device` block" to identify which positions on your instrument have one. |
| `command` | `expression-capable string` | — | Yes | `"Initialize"` | Command to execute. See enumeration below. |

### Temperature Parameters (for `"Incubate"` command)

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `use Temp` | `boolean` | — | No | `false` | If `true`, set temperature to target. Engine tolerates omission (default `False`). |
| `temp Set Point` | `expression-capable string` | °C | Conditional | `""` | Target temperature. Required when `use Temp` is `true`. Must be numeric with ≤1 decimal place, within device range. |
| `total Time` | `expression-capable string` | seconds | Conditional | `""` | Duration of incubation. Required when `Command="Incubate"`; ignored otherwise. Must be integer within device range. |

### Shake Parameters (for shaker devices with `"Incubate"` or `"Start Shaking"`)

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `use Shake` | `boolean` | — | No | `true` | If `true`, shake during incubation. Only applies to shaker devices. Engine tolerates omission (default `True`). |
| `shake RPM` | `expression-capable string` | RPM | Conditional | `""` | Shake speed. Required when shaking. Must be within device RPM range. |
| `shake Style` | `expression-capable string` | — | Conditional | `""` | Shake pattern. Required when shaking in single-style mode. |
| `continue Shaking` | `boolean` | — | Yes (when shaking) | `true` | If `true`, shaking continues after incubation until device is opened. Required whenever shaking is active (`Command="Start Shaking"`, or `Command="Incubate"` with `use Shake` true). |
| `deluxe Shake` | `boolean` | — | No | `false` | If `true`, enable dual-phase shaking with two alternating styles. Engine tolerates omission. |
| `shake Enabled` | `boolean` | — | No | `false` | UI-only flag reflecting whether the device supports shaking. Engine tolerates omission. Preserve for round-trip fidelity. |

### Dual-Phase Shake Parameters (when `deluxe Shake` is `true`)

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `shake Style1` | `expression-capable string` | — | Yes | `""` | First shake pattern. |
| `shake Time1` | `expression-capable string` | seconds | Yes | `""` | Duration of first shake phase. |
| `shake Style2` | `expression-capable string` | — | Yes | `""` | Second shake pattern. |
| `shake Time2` | `expression-capable string` | seconds | Yes | `""` | Duration of second shake phase. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

## Enumerated / Constrained Values

### Command Values

| Value | Applies To | Description |
|-------|-----------|-------------|
| `"Initialize"` | All | Initialize the device |
| `"Incubate"` | All | Temperature control with optional shaking |
| `"Start Shaking"` | Shaker only | Begin continuous shaking |
| `"Stop Shaking"` | Shaker only | Stop shaking |

### Shake Style Values

| Value |
|-------|
| `"Orbital (counter-clockwise)"` |
| `"Orbital (clockwise)"` |
| `"Diagonal (NE to SW)"` |
| `"Diagonal (NW to SE)"` |
| `"Vertical"` |
| `"Horizontal"` |

### Device Type Ranges

| Parameter | Older Models | SingleTEC |
|-----------|-------------|-----------|
| Min RPM | 100 | configurable |
| Max RPM | 2000 | configurable |
| Min Shake Time | 5 s | 1 s |
| Max Shake Time | 100 s | 99999 s |
| Min Total Time | 1 s | 1 s |
| Max Total Time | 999999 s | 99999 s |
| Temp Range | device-specific | device-specific |

## Cross-Field Validation Rules

1. `position` **must** exist on the deck and have an associated INHECO device.
2. `command` **must** be valid for the device type (static vs. shaker).
3. For `"Incubate"`: `total Time` must be an integer within the device's time range.
4. When `use Temp` is `true`: `temp Set Point` must be numeric (≤1 decimal), within device temp range.
5. When shaking: `shake RPM` must be within [minRPM, maxRPM].
6. When shaking in single mode: `shake Style` must be a valid style string.
7. When `deluxe Shake` is `true`: all four dual-phase parameters required and within ranges.

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

Incubate at 37°C for 5 minutes:

```json
{
  "stepType": "INHECO Peltier",
  "parameters": {
    "position": "P7",
    "command": "Incubate",
    "use Temp": true,
    "temp Set Point": "37.0",
    "total Time": "300"
  }
}
```

Incubate with shaking:

```json
{
  "stepType": "INHECO Peltier",
  "parameters": {
    "position": "P7",
    "command": "Incubate",
    "use Temp": true,
    "temp Set Point": "25.0",
    "total Time": "600",
    "use Shake": true,
    "shake RPM": "500",
    "shake Style": "Orbital (counter-clockwise)"
  }
}
```

Initialize device:

```json
{
  "stepType": "INHECO Peltier",
  "parameters": {
    "position": "P7",
    "command": "Initialize"
  }
}
```

## Common Mistakes

- **Authoring on an i3 method** — the step is registered for i5/i7 hardware and is not available on an i3 method.
- **Shake commands on a static peltier** — `"Start Shaking"`/`"Stop Shaking"` (and shake parameters under `"Incubate"`) require a shaker device; ranges and validity flip with device model.
- **Assuming older-model ranges on a SingleTEC** — RPM and Total Time bounds differ substantially (see §Device Type Ranges); validate against the actual device.
- **`temp Set Point` with more than one decimal place** — numeric validation limits it to ≤1 decimal (e.g., `"37.05"` is rejected).
- **Setting `deluxe Shake` without all four dual-phase keys** — `shake Style1`, `shake Time1`, `shake Style2`, `shake Time2` are all required together.
- **Omitting `continue Shaking` whenever shaking is active** — with `"Start Shaking"`, or `"Incubate"` while `use Shake` is `true`, this key must be present. Unlike `use Temp`/`use Shake`/`deluxe Shake`, omission is not tolerated: it errors at enqueue rather than defaulting to `true`.
