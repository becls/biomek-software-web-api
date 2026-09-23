# Introduction to Biomek and Liquid Handling Method Writing

This document introduces the Biomek instrument family, its pod types, the tree structure of a method, and the vocabulary the rest of the docs use. Read it before writing or editing a method.

> **Coverage note**: The [step documentation index](step-documentation/index.md) is the complete list of steps this documentation set covers, and [`add-on-steps/`](add-on-steps/index.md) covers the documented add-on steps. A step type outside those two lists may carry structural or parameter requirements that are not described here — obtain its parameter reference rather than inferring one.

---

## The Biomek Instrument

A **Biomek** is an automated liquid handling workstation. It is a robotic platform that moves precise volumes of liquid between containers (wells, tubes, reservoirs) on a work surface called a **deck**.

Biomek instruments are used in life science laboratories to automate repetitive pipetting tasks — serial dilutions, plate replication, normalization, PCR setup, ELISA preparation, sample reformatting, and more.

### Instrument Variants

| Model | Chassis / Deck | Pod Configuration | Gripper |
|-------|---------------|-------------------|---------|
| **i3** | One arm, 15 deck positions (default layout) | Fixed-8 pod (integrated) | Integrated with pod |
| **i5** | One arm, up to 25 deck positions | One pod: Span-8 **or** Multichannel | Rotating gripper |
| **i7** | One or two arms (one per pod), up to 40 deck positions | One or two pods: any combination of Span-8 and/or Multichannel ("dual-pod" when two) | Rotating gripper |

On an i7 each pod has its own arm, so a two-pod ("dual-pod") i7 is a genuine two-arm system. Default i3 decks often leave a column open — the i3's leftmost column is typically reserved for add-ons and a trash/storage area — so a default layout can show fewer positions than the deck can physically hold.

**Pod naming.** In the `pod` field of any step, use the pod's **configured name**, which comes from the instrument's pod settings rather than from the pod's type. Do **not** invent names like `"MPod"`, `"MC"`, or `"Span8"` — the instrument cannot substitute a made-up pod name, and enqueue fails.

The name follows one rule: **`PodN` is a 1-based index of the pods, counted left→right**, so the leftmost pod is `"Pod1"`, and the one to its right is `"Pod2"`. It is a *position* index, not a pod type — a single-pod instrument's only pod is always `"Pod1"`, and on a two-pod i7 the left pod is `"Pod1"` and the right pod is `"Pod2"`. Which type sits at which index is defined in the instrument's pod settings:

| Instrument | `"Pod1"` | `"Pod2"` |
|------------|----------|----------|
| i3 | Fixed-8 | — |
| i5 Multichannel | Multichannel | — |
| i5 Span-8 | Span-8 | — |
| i7 Multichannel | Multichannel | — |
| i7 Span-8 | Span-8 | — |
| i7 Hybrid | Multichannel | Span-8 |
| i7 Dual-Multichannel | Multichannel | Multichannel |

Some methods parameterize the pod name through a `let`/global expression (`"=podID"`, `"=thisSpanPod.name"`) instead of a literal; this is legitimate whenever the enclosing scope defines the referenced variable.

One caution: **do not treat `"Pod2"` as the canonical Span-8 pod.** Span-8 examples using `"Pod2"` reflect where the Span-8 sits on an i7 Hybrid — not a rule. On a standalone Span-8 the runnable value is `"Pod1"`. Always match the target instrument's actual configuration by reading the pod name from the instrument's pod settings.

### Pod Types

- **Fixed-8**: 8 pipetting channels in a column, all move together on shared X, Y, Z, and D (syringe) axes. Integrated gripper for moving labware. Uses disposable tips. Fixed-8 uses only manually specified pipetting techniques — auto-select is not supported for Fixed-8 aspirate/dispense/mix steps.
- **Span-8**: 8 independent probes with variable spacing. Each probe has independent Z and syringe (D) axes. Supports liquid-level sensing (LLS). Rotating gripper. Tip type is configured **per group of four** probes — configurations include fixed tips (always attached) or disposable tips (loaded from boxes).
- **Multichannel (MC)**: Interchangeable heads that pipette many wells at once in an SBS-aligned grid. Available densities are 96-channel and 384-channel. All channels share one syringe axis. Rotating gripper. Uses disposable tips loaded from a tip-loader ALP (a permanent deck fixture designed for MC tip loading). A 96-channel head on a 384-well plate accesses one quadrant of 96 wells at a time; use the Multichannel Select Tips container family for partial-mandrel operations.

The Multichannel pod supports three types of heads, which cannot be changed at runtime:
- 300 μL MC-96 Head
- 1200 μL MC-96 Head
- 60 μL MC-384 Head

96-channel heads can access 384-well labware one quadrant at a time.

Pipetting cannot exceed the volume capacity of the pod (Fixed-8), head (MC),
syringe (Span-8), or tip, whichever is smaller. Refer to the Biomek i5 and
Biomek i7 Instructions for Use and the Biomek i3 Instructions for Use for
information about tip capacities (B54473 and D24406).

### Choosing a pod for a given layout

- **Multichannel (MC)** moves all channels together on a fixed grid. Use it when you address **the whole plate**. To touch a *subset* of wells (a single row, an arbitrary set, fewer than a full column) you must add a **Multichannel Select Tips** step first — an MC Aspirate/Dispense on its own cannot pick an arbitrary well pattern.
- **Span-8** has 8 independently addressable probes. Use it for **arbitrary or partial** well sets (e.g. 12 samples that don't fill full columns), for per-well volume differences, and for reagent/reservoir work.
- **Fixed-8** behaves like a fixed 8-probe column; see the Fixed-8 step docs.
- A `Transfer`/`Combine` step runs on whichever pod its resolved `pod` names — so the same layout question decides the `pod` value.
- Only choose a pod that is present on the given instrument configuration.

---

## Core Concepts

### Deck

The deck is the physical work surface of the instrument. It is divided into **positions** — defined slots where labware sits. Each position has a known location in the instrument's coordinate system. Do not guess at deck position names — ask the user or read the corresponding local deck file if available.

### Labware

Labware is any container placed on the deck: microplates (96-well, 384-well, 1536-well), reservoirs, tube racks, tip boxes, waste containers. Each labware item has a **labware class** that defines its geometry (well shape, spacing, depth, per-well capacity). Do not guess at available labware — ask the user or read the corresponding local project file if available.

### Tips

Biomek instruments pipette using disposable tips (or fixed tips on some Span-8 configurations). Tips are loaded from tip boxes on the deck (or from a tip-loader ALP for Multichannel). The tip type determines the volume range and some physical properties of pipetting. For a pod that uses disposable tips, tips must be loaded before any pipetting is performed. At the end of a method, it is best practice to unload any disposable tips still loaded. Do not guess at available tip types — ask the user or read the corresponding local project file if available.

Certain operations require certain tip types. LLS and Clot Detection require conductive tips or fixed tips. Piercing requires piercing fixed tips.

### Techniques and Templates

A **pipetting template** is a project entity that defines the physical procedure the instrument executes for an aspirate, dispense, or mix operation — as an ordered sequence of actions (aspirate, dispense, tip touch, air gap, etc.).

A **pipetting technique** is a separate project entity that stores parameter values (heights, speeds, delays, blowout, tip touch, prewet, LLS, clot detection, etc.) and is *associated* with a template. Together they control how any aspirate, dispense, mix, pod height, pod speed, or tip touch operation is performed.

Some pods (Span-8, Multichannel) can auto-select a technique from context (tip type, labware class, volume range, liquid type). Fixed-8 requires a technique named explicitly.

For detail, see [Pipetting Techniques, Templates, and Auto-Selection](Pipetting-Techniques-and-Templates.md).

---

## Methods

A **method** is a program that tells the Biomek what to do. It is structured as a **tree of steps** — a hierarchy where some steps contain other steps as children.

When the user presses "Run," the Biomek software walks through this tree top-to-bottom, executing each step in sequence. Container steps (groups, loops, conditionals) control flow by holding child steps that execute under their rules.

### Method Tree Structure

Think of a method like an outline. Note that low-level pipetting steps are **pod-prefixed** — there is no bare `"Aspirate"`, `"Dispense"`, or `"Mix"` at method level. `Transfer` and `Combine` resolve their pod family at run time from the `pod` field.

```
Method
├── Start                        (always first — boundary marker)
├── Instrument Setup             (deck / labware / tip layout)
├── Transfer                     (aspirate from source, dispense to destination)
├── Group                        (contains a sequence of steps)
│   ├── Span-8 Aspirate
│   ├── Span-8 Dispense
│   └── End                      (terminator — marks end of Group)
├── Loop                         (repeats its children N times)
│   ├── Multichannel Aspirate
│   ├── Multichannel Dispense
│   └── End                      (terminator)
├── If                           (conditional branch)
│   ├── Named Container (Then)
│   │   ├── Transfer
│   │   └── End
│   └── Named Container (Else)
│       ├── Move Labware
│       └── End
└── Finish                       (always last — boundary marker)
```

`Named Container` is a step type. `If` always has exactly two of them as children, representing the Then and Else branches.

---

## Common steps

The steps you'll reach for most often are **Start**/**Finish** (method boundaries), **Instrument Setup**, the transfer-family steps **Transfer**/**Combine**, the per-pod pipetting families (**Fixed-8**, **Span-8**, **Multichannel** aspirate/dispense/mix/load/unload), and the control-flow containers **Group**/**Loop**/**If**/**Let**. For the complete catalog and structural rules, see the [step documentation index](step-documentation/index.md).

> **Add-on steps.** Beyond this core set, add-on modules and device integrations register their own step types — sometimes under a COM ProgID rather than a friendly name. These are documented separately in [`add-on-steps/`](add-on-steps/index.md); see also *Beyond the Built-in Steps* in [Method JSON Structure](Method-JSON-Structure.md).

---

## How Users Build Methods in the Biomek Editor

In the Biomek Editor (the desktop GUI application), a new method already contains `Start` and `Finish`. Users drag step types from a toolbox and drop them between those two anchors, then configure each one through a step-specific dialog — source and destination labware, volumes, tip handling. Steps that belong together go inside containers (groups, loops, if-branches) for control flow.

The editor enforces structural rules automatically: terminators are added when a container is created, anchors cannot be moved or crossed, and `If` always gets its Then/Else branches on creation. Hand-authored JSON must satisfy the same rules.

---

## Key Terminology Quick Reference

Terms used across these documents.

| Term | Meaning |
|------|---------|
| **Method** | A program (tree of steps) that automates liquid handling on a Biomek |
| **Step** | A single node in the method tree — either a leaf (action) or container (control flow) |
| **Deck** | The work surface with defined positions for labware |
| **Labware** | Physical containers (plates, reservoirs, tip boxes) placed on deck positions |
| **Position** | A defined slot on the deck where labware can be placed |
| **ALP** | Automated Labware Positioner — a bolted deck fixture (tip loader, wash station, trash, static holder) |
| **Tip** | Disposable (or fixed) pipette tip used for liquid transfer |
| **Template** | A project entity: an ordered sequence of actions defining the physical procedure of an aspirate/dispense/mix |
| **Technique** | A project entity: parameter values (heights, speeds, delays) associated with a template |
| **Pod** | The pipetting head assembly (Fixed-8, Span-8, or Multichannel) |
| **Instrument Setup** | Configuration step that defines which labware/tips are on which positions |
| **Anchor** | Start or Finish — immovable boundary markers at method root; other top/bottom anchors exist for some containers |
| **Terminator** | The immovable final child marking a container's end — `End`, `Multichannel Select Tips End`, or a container-specific terminator registered by an add-on step |
| **Named Container** | A step type used as a labeled branch inside `If` |
| **Enqueue** | The phase you enter by pressing Run or Simulate: the method tree is compiled into the actions the instrument will perform, and most configuration errors surface here, before anything moves |
| **Execute** | The phase after enqueue, in which the compiled actions drive the physical hardware |

---

## What "Writing a Method" Means

When you write or generate a Biomek method, you are constructing the **step tree** — the hierarchical structure of steps with their parent-child relationships and (optionally) their configuration parameters.

At minimum, a valid method must have:

1. A root node representing the method.
2. `Start` as the first child of the root.
3. `Finish` as the last child of the root.
4. Any operational steps inserted between the anchors — typically starting with `Instrument Setup` to define what labware and tips are on the deck.
5. All containers properly terminated.
6. All structural rules satisfied (see [Method JSON Structure](Method-JSON-Structure.md) for the full rule set).

The **step configuration** (which wells, which volumes, which labware, which technique) lives in each step's dictionary — that is a separate concern from the tree structure. See per-step reference in [step-documentation/](step-documentation/).

---

## Common Method Patterns

### Simple Transfer
Aspirate from a source plate, dispense to a destination plate. One `Transfer` step between the anchors.

### Serial Dilution
Prefer the dedicated `Span-8 Serial Dilution` or `Multichannel Select Tips Serial Dilution` steps when they fit the pod. If you need finer control, fall back to a `Loop` containing a `Transfer` that increments the source/destination column each iteration.

### Conditional Processing
An `If` where the Then branch (a `Named Container`) processes samples one way and the Else branch (a `Named Container`) handles the alternative path (e.g., skip low-concentration samples).

### Multi-Step Protocol
A `Group` (or `Let` for scoped variables) containing multiple sequential operations — aspirate reagent, mix, incubate with a `Pause`, transfer product.

### Plate Replication
`Transfer` configured to copy all wells from a source plate to one or more destination plates.

### Partial-Plate Multichannel
Wrap a `Multichannel Select Tips` container around `Multichannel Select Tips Load` / `Multichannel Select Tips Aspirate` / `Multichannel Select Tips Dispense` / `Multichannel Select Tips Unload` children when only some mandrels of an MC head should be active.
