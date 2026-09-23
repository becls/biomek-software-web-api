# Run Program

| Property | Value |
|----------|-------|
| stepType | `"Run Program"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | All |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).

## Behavior Summary

The Run Program step launches an external program during method execution. `executableName` names the program, `parameters` supplies its command line, `runPath` sets its working directory, `visibility` sets the window show-mode, and `resource` names a resource that is reserved before the program starts.

`waitForRuntime` selects one of three wait modes (see Enumerated Values): continue immediately, hold the resource until the program exits, or block all method activity including look-ahead. When `captureOut` is `true`, the program's standard output is written to the global variable named by `globalName`. When `lightCurtain` is `true`, the light curtain/interlock on the system is paused — so it can be broken — while the program runs, and resumed when the program returns.

Every string-valued key is expression-evaluated at enqueue; `lightCurtain`, `isInteractive` and `captureOut` are read as booleans.

## Parameters Reference Table

### Step-Level Keys

The step defaults all 11 keys, so exported methods always carry all of them.

Every key below except `timeEstimate` is read at enqueue without a default: omitting one raises a missing-key SYSTEM ERROR at Run time (see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md#0-what-required-means-in-the-parameters-table) §0). Unprefixed string values are literal; prefix with `=` to evaluate (e.g. `"=\"bsf shake \" & shakeSpeed"`).

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `executableName` | `expression-capable string` | — | **Yes** | `""` | Path/name of the program to run. If it evaluates to empty, enqueue fails as a file-required error. |
| `parameters` | `expression-capable string` | — | **Yes** | `""` | Command-line arguments passed to the program. *(This parameter key literally spells `parameters`, distinct from the step node's `parameters` object.)* |
| `runPath` | `expression-capable string` | — | **Yes** | `""` | Working directory the program starts in. |
| `resource` | `expression-capable string` | — | **Yes** | `"<Everything>"` | Resource to reserve before running — see Enumerated Values for the three forms the editor writes. A value that does not evaluate to a resource object reserves the universal resource instead. |
| `visibility` | `expression-capable string` | — | **Yes** | `"Invisible"` | Window show-mode; mapped to a Win32 `SW_*` code. The **evaluated** value must be one of the tokens in Enumerated Values. |
| `waitForRuntime` | `expression-capable string` | — | **Yes** | `"Hold the resource until the program completes"` | How the method waits. **Values are full English sentences** — see Enumerated Values; the constraint applies to the evaluated value. |
| `timeEstimate` | `expression-capable string` | seconds | No | `"1"` | Estimated run time, used for scheduling. The only key with a fallback, so omission is safe. The evaluated value is converted to a whole number of seconds — author an integer string (e.g. `"1"`, `"30"`), or an expression that evaluates to a number (`"=Incubation_Timer"`); a value that cannot be converted raises a translation error (see Cross-Field Validation). |
| `lightCurtain` | boolean | — | **Yes** | `false` | If `true`, the light curtain/interlock **is paused** — so it can be broken — while the program runs. See Runtime notes. |
| `isInteractive` | boolean | — | **Yes** | `false` | Marks the launched action as requiring user interaction. |
| `globalName` | `expression-capable string` | — | **Yes** | `""` | Name of the global variable that receives captured standard output. Required (non-empty after evaluation) whenever `captureOut` is `true`, in **every** wait mode — otherwise enqueue fails as a global-name-required error. The global is created if it does not already exist, and later steps read it as an expression (`"=BarcodeResult"`). The name must be a valid identifier: begin with a letter, then letters, digits or underscores only. See [Set Global](69-Set-Global.md) for the global-variable model. |
| `captureOut` | boolean | — | **Yes** | `false` | Capture the program's standard output into the global named by `globalName`. The `globalName` requirement is enforced in every wait mode, but real output is only delivered in the block-all mode — see Runtime notes. |

Universal base keys (`caption`, `disabled`, …) — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md). Do not author them.

**Exception — `dynamic?` is authored on Run Program.** Unlike most steps, the editor writes `dynamic?` explicitly on this step: `true` when `captureOut` is `true` or `waitForRuntime` is the block-all sentence, else `false`. Author it to match the same rule when writing a Run Program step from scratch, so the method round-trips identically and the error-handling behavior matches an editor-saved step.

## Enumerated / Constrained Values

### waitForRuntime Values

Full English sentences used as enum keys. Only these three select a wait behavior:

| Value | Effect |
|-------|--------|
| `"Allow the method to continue independently"` | Fire-and-forget: the method continues immediately and nothing is held. |
| `"Hold the resource until the program completes"` *(default)* | Resource held until the program exits. Subsequent method steps that do not depend on the given resource continue running. |
| `"Block all method activity including look-ahead until the program completes"` | Blocks method activity; also the only mode in which `captureOut` delivers real output. |

An unrecognized value is **not** rejected: it selects neither wait nor block, so the step behaves exactly like `"Allow the method to continue independently"`.

### resource Values

The editor writes one of three forms; author the same forms in JSON.

| Value | Meaning |
|-------|---------|
| `"<Everything>"` *(default)* | Magic string — the angle brackets are a **literal** value, not a placeholder. Reserves the universal resource, so nothing else runs concurrently. |
| `"=Positions.<DeckPositionName>"` | Reserve a single deck position, e.g. `"=Positions.BSF"`. This is the common case. |
| `"=World.Devices.Pipettor1.Pod1"` / `"=World.Devices.Pipettor1.Pod2"` | Reserve a pod. |

A bare position name such as `"P4"` is **not** a resource reference — unprefixed values are literal strings, and a value that does not evaluate to a resource object silently falls back to reserving the universal resource. Write `"=Positions.P4"` instead. If the expression itself fails to evaluate, the step fails at enqueue — see Cross-Field Validation.

### visibility Values

`"Invisible"` (default, → `SW_HIDE`), `"Visible"` (→ `SW_SHOWNORMAL`), or a raw ShowWindow constant: `"SW_HIDE"`, `"SW_MAXIMIZE"`, `"SW_RESTORE"`, `"SW_SHOW"`, `"SW_SHOWDEFAULT"`, `"SW_SHOWMAXIMIZED"`, `"SW_SHOWMINIMIZED"`, `"SW_SHOWMINNOACTIVE"`, `"SW_SHOWNA"`, `"SW_SHOWNOACTIVATE"`, `"SW_SHOWNORMAL"`. Any other value raises a visibility-flag error.

## Cross-Field Validation Rules

The rules the enqueue-time validation enforces:

1. **`executableName` required.** If it evaluates to empty, enqueue fails as a file-required error.
2. **`timeEstimate` must parse as an integer.** On expression-eval failure, enqueue fails as a translation error.
3. **`captureOut` requires `globalName`.** If `captureOut` is `true` and `globalName` evaluates to empty, enqueue fails as a global-name-required error. This check runs in **every** wait mode, not just the block-all mode.
4. **`resource` must evaluate.** A `resource` expression that fails to evaluate fails the step at enqueue as an invalid-resource error. Literal values (`<Everything>`, `""`, or a plain string) never reach this check, because a literal always evaluates successfully.
5. **`visibility` must be a known flag.** An unrecognized value fails as an invalid-visibility-flag error.
6. **`globalName` must be a valid identifier.** When `captureOut` is `true`, writing the global rejects a name that does not begin with a letter or that contains anything but letters, digits and underscores.

Runtime notes:

- `lightCurtain: true` reserves the universal resource — everything else must be
  idle — then pauses the light curtain/interlock for the duration of the
  program and resumes it when the program returns, including on error.
- When `captureOut` is `true`, the program's standard output is written to the global named by `globalName`. While simulating, that global is set to the literal `"SIMULATING"` instead. Real output is only delivered when `waitForRuntime` is the block-all sentence; in the other two wait modes the step still writes the global, but it receives an empty string.

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

Fire-and-forget launch, invisible window:

```json
{
  "stepType": "Run Program",
  "parameters": {
    "executableName": "C:\\Tools\\notify.exe",
    "parameters": "--plate done",
    "runPath": "",
    "resource": "<Everything>",
    "visibility": "Invisible",
    "waitForRuntime": "Allow the method to continue independently",
    "timeEstimate": "1",
    "lightCurtain": false,
    "isInteractive": false,
    "globalName": "",
    "captureOut": false,
    "dynamic?": true
  }
}
```

Blocking run that captures stdout into a global:

```json
{
  "stepType": "Run Program",
  "parameters": {
    "executableName": "C:\\Tools\\readbarcode.exe",
    "parameters": "",
    "runPath": "C:\\Tools",
    "resource": "<Everything>",
    "visibility": "Invisible",
    "waitForRuntime": "Block all method activity including look-ahead until the program completes",
    "timeEstimate": "10",
    "lightCurtain": false,
    "isInteractive": false,
    "globalName": "BarcodeResult",
    "captureOut": true,
    "dynamic?": true
  }
}
```

Expression-driven call reserving one deck position:

```json
{
  "stepType": "Run Program",
  "parameters": {
    "executableName": "C:\\Biomek\\shaker.py",
    "parameters": "=\"bsf shake \" & shakeSpeed",
    "runPath": "",
    "resource": "=Positions.BSF",
    "visibility": "Invisible",
    "waitForRuntime": "Hold the resource until the program completes",
    "timeEstimate": "=Incubation_Timer",
    "lightCurtain": false,
    "isInteractive": false,
    "globalName": "",
    "captureOut": false,
    "dynamic?": true
  }
}
```

## Common Mistakes

- **Wrong `waitForRuntime` value.** Use one of the three sentences verbatim. An unrecognized value is not rejected — the step silently falls through to fire-and-forget (no wait, no block).
- **`timeEstimate` as a JSON number.** Source stores it as a string and re-evaluates it as an expression; keep it a string (`"5"`).
- **Treating `<Everything>` as a placeholder.** It is a literal magic value; leaving it as-is reserves the universal resource.
- **A bare position name in `resource`.** `"P4"` is a literal string, not a resource reference, and quietly reserves the universal resource. Write `"=Positions.P4"`.
- **Expecting captured output outside the block-all mode.** `globalName` is required whenever `captureOut` is `true` — in every wait mode — but the global only receives real output when `waitForRuntime` is the block-all sentence.
- **`lightCurtain: true` misread as "enforce".** It *pauses* the curtain so it may be broken during the program — the opposite of what the name suggests.
