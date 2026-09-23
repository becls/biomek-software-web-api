# Step Documentation Catalog

The Step Reference has one file per step. Each per-step file has a header table
(`stepType`, `Category`, `Compatible Hardware`), a Behavior Summary, a
Parameters Reference, enumerated values, cross-field rules, and worked JSON
examples.

Refer to the Biomek Software Reference Manual (B56358) for more information
about steps and their usage.

This index is a **navigation aid** — a single catalog grouped by pod family / area, with one row per step showing its compatible hardware and a one-line summary. For definitive detail on any step, open its own file. For the structural rules (Leaf / container / terminator / anchor), see [Method JSON Structure](../Method-JSON-Structure.md).

> **Only author the steps listed here.** Other step types exist internally (framework- or add-on-only) but are not intended for hand-authoring in the JSON format. Steps contributed by add-on modules and device integrations are documented separately — see [`../add-on-steps/`](../add-on-steps/index.md).

Shared conventions (universal base keys, expression syntax, labware/position resolution, probe/mandrel selection, the variable environment) live in [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md) and in [`../concept-guides/`](../concept-guides/index.md). Every per-step doc assumes you have read those.

---

## Step catalog (by pod family / area)

Grouped by what the step operates on — pod hardware family, deck/setup activity, or method-structure concern. A step that participates in more than one grouping (e.g., a pod pipetting step also relevant to a broader category) is listed under its primary use.

### Any-pod pipetting

Consolidated pipetting steps that decide the pod family internally.

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 11 | [Transfer](11-Transfer.md) | i5, i7, i3 | Moves liquid from a single source location to one or more destinations, including tip handling. |
| 12 | [Combine](12-Combine.md) | i5, i7 | Mirror image of Transfer: moves liquid from multiple sources into a single destination, including tip handling. |

### Fixed-8 pipetting (i3)

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 36 | [Fixed-8 Aspirate](36-Fixed-8-Aspirate.md) | i3 | Aspirates from a labware position using the Fixed-8 pod; technique must be specified (no auto-select). |
| 37 | [Fixed-8 Dispense](37-Fixed-8-Dispense.md) | i3 | Delivers liquid to a labware position using the Fixed-8 pod; technique must be specified (no auto-select). |
| 38 | [Fixed-8 Mix](38-Fixed-8-Mix.md) | i3 | Mixes liquid in place at a labware position using the Fixed-8 pod, aspirating and dispensing a set volume per cycle. |
| 39 | [Fixed-8 Load Tips](39-Fixed-8-Load-Tips.md) | i3 | Picks up disposable tips from a 96-density tip box and attaches them to the Fixed-8 pod. |
| 40 | [Fixed-8 Unload Tips](40-Fixed-8-Unload-Tips.md) | i3 | Removes tips from the Fixed-8 pod — return to box, discard to trash, or place at a specific location. |

### Span-8 pipetting

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 13 | [Span-8 Transfer From File](13-Span-8-Transfer-From-File.md) | Span-8 pod (i5, i7) | Reads a comma-delimited pick-list file and builds one well-to-well aspirate/dispense per row on a Span-8 pod. |
| 14 | [Span-8 Serial Dilution](14-Span-8-Serial-Dilution.md) | Span-8 pod (i5, i7) | Performs a serial dilution across a contiguous run of wells on a Span-8 pod, expanding internally into load/aspirate/dispense/transfer/wash operations. |
| 15 | [Span-8 Aspirate](15-Span-8-Aspirate.md) | Span-8 pod (i5, i7) | Aspirates from source labware using one or more selected probes on a Span-8 pod. |
| 16 | [Span-8 Dispense](16-Span-8-Dispense.md) | Span-8 pod (i5, i7) | Delivers liquid from Span-8 tips into destination labware. |
| 17 | [Span-8 Load Tips](17-Span-8-Load-Tips.md) | Span-8 pod (i5, i7) | Loads disposable tips onto the selected Span-8 probes. |
| 18 | [Span-8 Unload Tips](18-Span-8-Unload-Tips.md) | Span-8 pod (i5, i7) | Discards disposable tips from the selected Span-8 probes into the trash. |
| 19 | [Span-8 Wash Tips](19-Span-8-Wash-Tips.md) | Span-8 pod (i5, i7) | Washes Span-8 tips (or fixed mandrels) at a wash station, in either passive or active mode. |

### Multichannel (fixed head) pipetting

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 20 | [Multichannel Aspirate](20-Multichannel-Aspirate.md) | i5 or i7 with a Multichannel pod | Aspirates from source labware using a fixed multichannel head (96- or 384-channel). |
| 21 | [Multichannel Dispense](21-Multichannel-Dispense.md) | i5 or i7 with a Multichannel pod | Dispenses into destination labware using a fixed multichannel head. |
| 22 | [Multichannel Load Tips](22-Multichannel-Load-Tips.md) | i5 or i7 with a Multichannel pod | Loads disposable tips onto the multichannel head from a matching tip box on the deck. |
| 23 | [Multichannel Unload Tips](23-Multichannel-Unload-Tips.md) | i5 or i7 with a Multichannel pod | Removes disposable tips from the multichannel head; no-op if none are loaded. |
| 24 | [Multichannel Mix](24-Multichannel-Mix.md) | i5 or i7 with a Multichannel pod | Aspirates and re-dispenses in place a set number of times to mix well contents on the multichannel head. |
| 25 | [Multichannel Wash Tips](25-Multichannel-Wash-Tips.md) | i5 or i7 with a Multichannel pod | Washes the tips on a multichannel head by mixing them in a wash-station reservoir. |

### Multichannel Select-Tips family

A container plus a specialized terminator plus a suite of leaf steps that operate on the currently-loaded partial tip pattern.

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 26 | [Multichannel Select Tips](26-Multichannel-Select-Tips.md) | Multichannel pod only | Container that groups partial-tip-pattern operations (aspirate, dispense, mix, load, unload, serial dilution, advanced load, advanced unload) on a multichannel pod. |
| 27 | [Multichannel Select Tips End](27-Multichannel-Select-Tips-End.md) | Multichannel pod only | Mandatory terminator for the Select Tips container; validates that no tips remain on the pod. |
| 28 | [Multichannel Select Tips Serial Dilution](28-Multichannel-Select-Tips-Serial-Dilution.md) | Multichannel pod only | Serial-dilution protocol using the multichannel pod's currently loaded tip pattern of equally spaced rows or columns. |
| 29 | [Multichannel Select Tips Aspirate](29-Multichannel-Select-Tips-Aspirate.md) | Multichannel pod only | Aspirates at the target labware using the multichannel pod's currently loaded tip pattern. |
| 30 | [Multichannel Select Tips Dispense](30-Multichannel-Select-Tips-Dispense.md) | Multichannel pod only | Dispenses at the target labware using the multichannel pod's currently loaded tip pattern; supports empty-tips mode. |
| 31 | [Multichannel Select Tips Mix](31-Multichannel-Select-Tips-Mix.md) | Multichannel pod only | Performs repeated aspirate/dispense cycles at one position to mix well contents, using the current tip pattern. |
| 32 | [Multichannel Select Tips Load](32-Multichannel-Select-Tips-Load.md) | Multichannel pod only | Picks up tips from a tip box in a specific pattern (single, rows, or columns), with an optional backup location. |
| 33 | [Multichannel Select Tips Unload](33-Multichannel-Select-Tips-Unload.md) | Multichannel pod only | Unloads tips from the multichannel pod to a tip box (auto-fit) or discards to trash. |
| 34 | [Multichannel Select Tips Advanced Load](34-Multichannel-Select-Tips-Advanced-Load.md) | Multichannel pod only | Loads tips from a specific row/column in a tip box for precise placement control. |
| 35 | [Multichannel Select Tips Advanced Unload](35-Multichannel-Select-Tips-Advanced-Unload.md) | Multichannel pod only | Unloads tips to a specific row/column in a tip box for precise placement control. |

### Method boundaries and deck setup

Steps that mark start/finish, configure the deck, or move labware and pods around the deck.

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 01 | [Start](01-Start.md) | All | Mandatory first step in every method; defines method-level variables, weak-bound defaults, and runtime prompts, and performs hardware initialization at run time. |
| 02 | [Finish](02-Finish.md) | All | Mandatory last step in every method; performs post-run cleanup (tips off, pods parked, deck state cleared) under independent boolean flags. |
| 03 | [Instrument Setup](03-Instrument-Setup.md) | All | Selects a named deck layout and populates positions with the labware in `deckItems`; optionally verifies the setup with the operator. |
| 04 | [Move Labware](04-Move-Labware.md) | All | Moves labware from one deck position to another using a pod-mounted gripper, with control over stack depth and grip side. |
| 05 | [Cleanup](05-Cleanup.md) | i5, i7 | Disposes of tips and optionally returns tip boxes to their configured destinations. |
| 06 | [Move Pod](06-Move-Pod.md) | All | Moves a pod to a specified deck position without pipetting — used to clear the pod before manual operations or a Pause. |
| 07 | [Hold Labware](07-Hold-Labware.md) | i5, i7 | Picks labware up with a pod gripper, holds it while child steps run, then places it back at the original source. |

### Control flow

Containers and leaf steps that shape the order, condition, or timing of what runs.

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 08 | [End](08-End.md) | All | Structural marker that closes a container step; the required last child of most container steps. |
| 49 | [Break](49-Break.md) | All | Terminates execution of one or more enclosing loops; must be inside a Loop to have effect. |
| 50 | [Comment](50-Comment.md) | All | Runtime no-op that carries description/comment text shown in the editor and in method reports. |
| 52 | [Group](52-Group.md) | All | Free-form container that enqueues its child steps in order — used to organize the tree. |
| 53 | [If](53-If.md) | All | Evaluates a script expression and executes exactly one of two branches (Then / Else Named Container children). |
| 54 | [Just In Time](54-Just-In-Time.md) | All | Free-form container whose child actions are scheduled as late as the schedule allows, rather than as early as possible. |
| 55 | [Worklist](55-Worklist.md) | All | Reads a CSV worklist file and repeats child steps once per row, binding the row's columns as scoped variables. |
| 58 | [Loop](58-Loop.md) | All | Repeats child steps a computed number of times from start/end/increment; optionally binds an iteration variable. |
| 59 | [Named Container](59-Named-Container.md) | All | Free-form container that groups child steps under a caption and adds no control flow. |
| 62 | [Pause](62-Pause.md) | All | Halts method execution until the operator acknowledges a prompt, or for a timed duration on a specific resource. |

### Data sets

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 41 | [Create Data Set](41-Create-Data-Set.md) | All (i3, i5, i7) | Reads a text file or SQL Server table and attaches the values as data sets on destination labware, one value per well. |
| 42 | [Data Set Management](42-Data-Set-Management.md) | All (i3, i5, i7) | Manipulates existing data sets on a piece of labware — copy, rename, remove, or change flags. |
| 43 | [Data Set Processing](43-Data-Set-Processing.md) | All (i3, i5, i7) | Applies a transformation expression to a source data set to build a new data set on the same labware. |
| 44 | [Data Set Reporting](44-Data-Set-Reporting.md) | All (i3, i5, i7) | Generates a report of every reportable data set on every piece of labware known to the world. |
| 45 | [View Data Sets](45-View-Data-Sets.md) | All (i3, i5, i7) | Design-time inspection for viewing data sets, labware properties, and global variables; runtime no-op. |

### Device integration

External and framework devices controlled from a method.

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 09 | [Device Action](09-Device-Action.md) | i5, i7 | Sends a command to a digital I/O device or generic script device attached to the instrument. |
| 10 | [INHECO Peltier](10-INHECO-Peltier.md) | i5, i7 | Controls INHECO thermal and shaking devices on the deck (temperature incubation, shaking, initialization). |
| 70 | [Run Program](70-Run-Program.md) | All | Launches an external program during method execution with a specified command line, working directory, window mode, and reserved resource. |
| 71 | [SILAS](71-SILAS.md) | All (requires configured SILAS modules) | Sends a command to an external SILAS-controlled device module (plate reader, incubator, centrifuge, barcode reader, etc.). |

### Scope, variables, and scripting

Steps that define patterns/variables, run scripts, or emit log lines.

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 51 | [Define Pattern](51-Define-Pattern.md) | All | Creates a named well-selection pattern (fixed, file-driven, or operator-prompted) usable by other steps. |
| 56 | [Let](56-Let.md) | All | Container that binds scoped variables (strong, weak, or prompted) for its child steps. |
| 67 | [Scripted Let](67-Scripted-Let.md) | All | Runs a script block, then enqueues child steps in a scope containing the variables the script bound. |
| 68 | [Script](68-Script.md) | All | Runs a script block at enqueue (during Run / Simulate); a leaf with no child steps. |
| 69 | [Set Global](69-Set-Global.md) | All | Defines or updates a global variable available to any step enqueued after it, at any nesting depth. |

### Procedures and methods

Sub-methods and reusable procedures called from a top-level method.

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 64 | [Run Method](64-Run-Method.md) | All | Runs another method from the current project inline as part of the current method, with optional variable overrides. |
| 65 | [Define Procedure](65-Define-Procedure.md) | All | Declares a named, reusable procedure made up of its own child steps; a later Run Procedure inlines the body. |
| 66 | [Run Procedure](66-Run-Procedure.md) | All | Invokes a procedure previously declared by Define Procedure, applying its own `let` values over the procedure's defaults. |

### Labware groups and iteration

Named labware groups and list-driven iteration.

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 60 | [Next Item](60-Next-Item.md) | All | Advances a global variable to the next value in a list; end-of-list behavior is chosen by `onDone`. |
| 72 | [Create Labware Group](72-Create-Labware-Group.md) | i5, i7 | Tags a set of labware as a named, ordered group so a later Next Labware step can cycle through them. |
| 73 | [Next Labware](73-Next-Labware.md) | i5, i7 | Cycles a named labware group to its next member — returns the current piece, brings the next unused one to the working location. |

### Fly-By Bar Code Reader

| # | Step | Compatible Hardware | What it does |
|---|------|---------------------|--------------|
| 46 | [Fly-By Read](46-Fly-By-Read.md) | i5, i7 | Uses a gripper-equipped pod to fly a stack of labware past a Fly-By Bar Code Reader, reading each piece. |
| 47 | [Fly-By Log](47-Fly-By-Log.md) | i5, i7 | Writes accumulated Fly-By Bar Code Reader read results to a CSV log file. |
