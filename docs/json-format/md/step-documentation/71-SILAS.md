# SILAS

| Property | Value |
|----------|-------|
| stepType | `"SILAS"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | All (requires configured SILAS modules on deck) |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).

## Behavior Summary

The SILAS step sends a command to an external SILAS-controlled device module (for example a plate reader, incubator, centrifuge, or barcode reader). It names the target module and hands it a pre-configured binary message carrying the command and its parameters. Set `dynamic?` to make the step wait for the action to finish; leave it off and the step is fire-and-forget.

Around that command the step also handles labware give/take between the deck and the device's internal storage, and manages light curtain/interlock access for pod clearance. It can create data sets from the data the device sends back.

The step is generic — a single step type handles all SILAS device types. Each configured SILAS module appears as a separate palette entry in the Biomek UI, but every one of them serializes to the same `"SILAS"` step type.

## Parameters Reference Table

### Core Parameters

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `module` | `string` | — | Yes | — | SILAS module name. Must name a SILAS module configured on the instrument. See [Deck Layouts Format](../biomek-file-formats/deck-layouts/deck-layouts-format-spec.md#the-device-block) §"The `device` block" for where configured module names come from. |
| `message` | `object` | — | Yes | — | SILAS command message. In JSON, serialized as a structured object (see Message Structure below). |
| `dynamic?` | `boolean` | — | No | `false` | If `true`, blocks until device action completes and enables RDD/DataSet features. |
| `lightCurtainAccess` | `boolean` | — | No | *(module default)* | If `true`, pauses pipettor light curtain/interlock and clears pod zones during action. Falls back to module's default if not bound. |

### Data Set Parameters (conditional: `dynamic?` must be `true`)

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `makeDataSet` | `boolean` | — | No | `false` | If `true`, create data set(s) from device response. |
| `dataSetName` | `string` | — | Conditional | `""` | Name for the created data set. The step has no built-in fallback name: when `dynamic?` and `makeDataSet` are both `true` and this is empty, enqueue fails as a data-set-name-required error. If the device returns more than one data message, the created data sets are named `<dataSetName>0`, `<dataSetName>1`, … |
| `copyOnTransfer` | `boolean` | — | No | `true` | If `true`, the created data sets travel with liquid transfers to the destination labware. |
| `translate` | `string` | — | No | *(key absent)* | COM ProgID of a custom translator object that interprets SILAS response messages. When present, the translator replaces the built-in data-set extraction (and `copyOnTransfer` is not applied). Omit the key entirely when no translator is in use: if the key is present but empty, enqueue fails with an error requiring the translator's ProgID. |

### Storage & Override Parameters

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `override Index` | `string` | — | No | — | Overrides the storage slot/stack at enqueue time; applied only when `Give Or Take` is not `"None"`. Writes the value into `Storage Model Info.Locations` under key `"1"` **and** injects it into `Command.Parameters.Index` (Array storage) or `Command.Parameters.Stack` (Stacks storage), according to the module's storage model. |
| `rdd` | `object` | — | No | *(auto-created)* | Runtime Data Dictionary. Injected into action context. Can contain `OtherProperties` list for simulation. |

### Display Parameters (design-time only)

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `defaultBitmap` | `string` | — | No | `""` | Bitmap path resolved from SILAS bitmap directory. |
| `defaultTooltip` | `string` | — | No | `""` | Tooltip from the SILAS message 'Action Description' field. |

> _Universal base keys (`caption`, `defaultCaption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md). Note: the step-specific `dynamic?` row above documents the SILAS blocking flag, which name-collides with the base key but has step-functional semantics on SILAS._

## Message Structure

In JSON serialization, the `message` parameter is converted from its binary format to a structured object. The logical structure is:

```
message:
  _biomekType: "SILASMessage"  — Type discriminator; must be exactly this value
  Command:                     — sub-message
    Command Name: string       — The SILAS command (device lifecycle "Open"/"Close"/"Initialize", or a module action)
    Parameters:                — sub-message; open-ended / command-specific
      Index: string            — (Array storage) slot index
      Stack: string            — (Stacks storage) stack name
      Get?: string             — "True"/"False"; written by the step on the auto-Open/Close messages
      Workflow: string         — "Transport" | "Pipette"
      Time: string             — Estimated time in MILLISECONDS; injected at enqueue from top-level Time x 1000 (do not author)
      ...                      — Other command-specific parameters (copied verbatim from the caller)
    Transports:                — sub-message injected at enqueue (do not author); keys are 1-based deck-position slot numbers as strings
      "1": string              — Slot number → labware UniqueID
  Time: string                 — Total estimated time in SECONDS (this is the one you author)
  Success?: boolean            — Device-reported success flag (captured, not authored); some devices emit the strings "True"/"False"
  Action Description: string   — Human-readable action tooltip
  Storage Model Info:          — sub-message; present for Give/Take
    Give Or Take: string       — "Give", "Take", or "None" (see the table below — the wording is from the module's point of view)
    Locations:                 — sub-message; keys are 1-based deck-position slot numbers as strings
      "1": string              — Value is a single string of the form `'<storage location>',<count>`, where the
                                 storage location is the module-side slot index (Array) or stack name (Stacks) —
                                 e.g. `'1',1` for slot 1, count 1
```

Messages captured from the SILAS device include additional device bookkeeping keys alongside the ones above (for example `%Address`, `%MessageID`, `%Time Stamp`, `Changed?`, `Touch Count`, and `Command.Action Summary` / `Command.Extensions`). Preserve them verbatim — round-tripping the containing Biomek method through Method JSON export/import keeps whatever the device supplied. Values are not restricted to strings: booleans and numbers round-trip as-is. A JSON array is only legal in one place — a two-element array `[value, submessage]` when the same key carries both a value and a sub-message.

Every key above is emitted **verbatim** and **case-sensitively**. Beyond `Time`/`Index`/`Stack`/`Get?`/`Workflow`, the contents of `Parameters` are command-specific and open-ended — do not assume a fixed schema.

> **Note**: The exact JSON shape of `message` depends on the SILAS module and action. The structure above shows the common fields.

## Enumerated / Constrained Values

### Give Or Take Values (inside Storage Model Info)

`Give` and `Take` are worded from the **module's** point of view, not the deck's — the module
gives labware out, or takes labware in. In the step editor these are the
"Retrieving From Module" and "Sending To Module" choices.

| Value | Editor wording | Meaning |
|-------|----------------|---------|
| `"Give"` | Retrieving From Module | The module gives labware out: transfer FROM the module's storage TO the deck position |
| `"Take"` | Sending To Module | The module takes labware in: transfer FROM the deck position TO the module's storage |
| `"None"` | No Change | No labware transfer |

### Storage Model Types (module property)

| Model | Access Pattern | Key in Message |
|-------|---------------|----------------|
| `Array` | Random access by integer index | `Command.Parameters.Index` |
| `Stacks` | LIFO access by stack name | `Command.Parameters.Stack` |

## Light Curtain AccessSide (module property, not a step key)

`AccessSide` and `AccessExtent` are configured on the SILAS module itself,
**not** the step. They govern the
pod-clearing behavior when `lightCurtainAccess` is `true`:

| `AccessSide` value | Pod Clearing Behavior |
|-------|----------------------|
| `"Left"` | Clears Pod1 from Min.X to Min.X + AccessExtent |
| `"Right"` | Clears the right-hand pod — Pod2 when the instrument has one, otherwise Pod1 — from Max.X − AccessExtent to Max.X |
| `"Center"` | Fully retracts Pod1 to its Min.X, and Pod2 (if present) to its Max.X |

The asymmetry is real: `"Left"` always clears Pod1, while `"Right"` picks Pod2 when the instrument
has one. Min.X and Max.X are the **selected** pod's own travel limits, so on a two-pod instrument
the `"Right"` range is measured against Pod2's maximum, not a deck-wide one.

`AccessSide` must be exactly `Left`, `Right`, or `Center` (matched case-insensitively). Failures
name the property and point at the Device Editor:

```
Access Side has not been defined. Check the settings in the Device Editor for this device.
Access Side value is invalid. Check the settings in the Device Editor for this device.
Access Extent has not been defined. Check the settings in the Device Editor for this device.
Negative values for "Access Extent" are not allowed. Check the settings in the Device Editor for this device.
An invalid value was found. Check the settings in the Device Editor for this device.
```

The last covers a non-numeric `AccessExtent`. Note that `AccessExtent` is validated **even when
`AccessSide` is `Center`**, which does not use it — the Device Editor grays the field out for
`Center` but still stores it, so this only bites a device definition edited outside the editor.

Set these in the SILAS module's configuration UI; do not add them to the step's `parameters`.

## Cross-Field Validation Rules

1. `module` **must** name a SILAS module configured on the instrument.
2. `message` **must not** be null and must be parseable (raises error with reason if malformed).
3. `message` **must** contain a `Command` submessage.
4. If `lightCurtainAccess` is `true`: module must have `AccessSide` and `AccessExtent` properties; `AccessExtent` must be ≥ 0.
5. If `makeDataSet` is `true`: `dataSetName` must not be empty.
6. For `"Give"` (module → deck): the module-side slot or stack must contain labware. With Array
   storage the destination deck position must also be empty.
7. For `"Take"` (deck → module): the source deck position must contain labware, and the module-side
   destination must have room — an empty slot for Array storage, or remaining capacity in the named
   stack for Stacks storage.
8. For Array storage: index must be in range 1..Items.Count.
9. For Stacks storage: the named stack must exist.

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

> The `module` name, `Command Name`, and the keys and values inside `Parameters` in the examples
> below are illustrative placeholders — substitute the actual names captured from the target SILAS
> device's UI or from a SILAS device message export. A real captured SILAS message also carries device
> bookkeeping keys (`%Address`, `%MessageID`, and so on) that are omitted here for readability; keep
> them when round-tripping the containing Biomek method through Method JSON export/import.

Basic SILAS action (fire-and-forget):

```json
{
  "stepType": "SILAS",
  "parameters": {
    "module": "PlateReader",
    "message": {
      "_biomekType": "SILASMessage",
      "Command": {
        "Command Name": "RunProtocol",
        "Parameters": {
          "ProtocolName": "Absorbance96"
        }
      },
      "Storage Model Info": {
        "Give Or Take": "None"
      },
      "Time": "30",
      "Action Description": "Run plate read protocol"
    },
    "dynamic?": false
  }
}
```

SILAS action that sends labware into the module and creates a data set. `"Take"` is correct here:
the module takes the plate in from deck position slot 1, into module slot 1 (Array storage).

```json
{
  "stepType": "SILAS",
  "parameters": {
    "module": "PlateReader",
    "message": {
      "_biomekType": "SILASMessage",
      "Command": {
        "Command Name": "ReadPlate",
        "Parameters": {
          "Index": "1"
        }
      },
      "Storage Model Info": {
        "Give Or Take": "Take",
        "Locations": {
          "1": "'1',1"
        }
      },
      "Time": "60",
      "Action Description": "Read plate with absorbance"
    },
    "dynamic?": true,
    "makeDataSet": true,
    "dataSetName": "AbsorbanceData",
    "lightCurtainAccess": true
  }
}
```

## Common Mistakes

- **Missing `dynamic?` when expecting data**: Data sets are only created when `dynamic?` is `true` — `makeDataSet` alone is silently ignored.
- Fabricating message content: SILAS messages are device-specific and should be captured from the SILAS UI editor. Do not invent message structures without knowledge of the target device's protocol.
- **Reversing `Give`/`Take`**: the words describe what the *module* does. `"Take"` sends labware from the deck into the module; `"Give"` retrieves labware from the module onto the deck. Picking the wrong one produces a labware error (empty source / occupied destination) rather than a silent mistake.
- Authoring `Command.Parameters.Time`: set the top-level `Time` (seconds) only. The step derives `Command.Parameters.Time` (milliseconds) from it at enqueue, so a hand-written value there is redundant and does not survive.
- Empty `dataSetName` with `makeDataSet`: there is no default name — enqueue fails as a data-set-name-required error.
- Empty `translate`: leave the key out when you have no translator object. Present-but-empty is treated as a misconfiguration and fails the step.
