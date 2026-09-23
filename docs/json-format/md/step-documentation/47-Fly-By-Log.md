# Fly-By Log

| Property | Value |
|----------|-------|
| stepType | `"Fly-By Log"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7 (companion to the Fly-By Bar Code Reader device; **not** i3) |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Envelope / structural rules: see [Method JSON Structure](../Method-JSON-Structure.md).

## Behavior Summary

The Fly-By Log step writes the accumulated Fly-By Bar Code Reader read results to a
comma-separated text log file with a header row: **Time, Plate Name, Initial Barcode, Final
Barcode, Recovery Action**. Each logged read records the time, plate name, initial bar code, final bar
code, and recovery action — the same fields the Fly-By Read step accumulates onto the device's
read list.

Place this step **after** one or more Fly-By Read steps that
populated the device's read list.

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `filename` | string (expression) | — | Yes | `""` | Full path to the output log file (typically `.txt`; the `.txt` convention comes from the editor's file-picker default and is not enforced at runtime). Must be non-blank; expression-evaluated at enqueue. |

Universal base keys (`caption`, node `disabled`, `dynamic?`, …) — see
[Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).

> This step persists only the key `filename`. Other FBBCR-related keys (e.g., a log-name key or
> device-level keys) are not read by this step — do not author them.

## Enumerated / Constrained Values

None — `filename` is a free-form path string (expression-capable).

## Cross-Field Validation Rules

1. `filename` must not be blank/empty.

## Structural Context

This step is a **leaf** and does not support authored `subSteps`.

## Canonical Example

```json
{
  "stepType": "Fly-By Log",
  "parameters": {
    "filename": "C:\\Users\\Public\\Documents\\BarcodeReads.txt"
  }
}
```

## Common Mistakes

- **Logging with no prior reads** — placing this step before any Fly-By Read produces an empty
  log; the device's `Reads` list is what gets emitted.
- **Authoring the wrong key name** — this step reads only the literal key `'filename'`; other
  logger-related keys do nothing here.
