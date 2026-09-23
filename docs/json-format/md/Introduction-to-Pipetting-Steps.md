# Introduction to Pipetting Steps

Reference information common to pipetting steps.

See [Pipetting Techniques and Templates](Pipetting-Techniques-and-Templates.md)
for more details, or refer to individual step documentation.

## Low-Level Pipetting Steps

Low-level pipetting steps consist of individual pipetting operations, such as
aspirate, dispense, or mix. Each pod type has its own set of low-level steps,
prefixed with the pod type.

Additionally, the Multichannel pod supports low-level pipetting steps which use
the whole head's worth of tips, as well as low-level pipetting tips with a
subset of tips loaded via the **Select Tips** steps.

The following table shows the step names for each low-level operation for each
pod type.

| Pod Type | Aspirate | Dispense | Mix | Wash Tips |
|----------|----------|----------|-----|-----------|
| Fixed-8 | Fixed-8 Aspirate | Fixed-8 Dispense | Fixed-8 Mix | (not supported) |
| Span-8 | Span-8 Aspirate | Span-8 Dispense | (not supported) | Span-8 Wash Tips |
| Multichannel | Multichannel Aspirate | Multichannel Dispense | Multichannel Mix | Multichannel Wash Tips |
| Multichannel Select Tips | Select Tips Aspirate | Select Tips Dispense | Select Tips Mix | (not supported) |

Note that Multichannel Select Tips steps must be contained in a **Select Tips**
group step. See [Multichannel Select
Tips](step-documentation/26-Multichannel-Select-Tips.md).

---

## High-Level Pipetting Steps

High-level pipetting steps put together sequences of multiple pipetting
operations, including tip handling between operations.

The following table shows the pod types supported by each high-level pipetting
step.

| Step Name | Description | Supported Pod Types |
|-----------|-------------|---------------------|
| Transfer | Transfer from the selected wells in a single source to the selected wells in multiple destinations. | Fixed-8, Span-8, Multichannel |
| Combine | Transfer from the selected wells of multiple sources to the selected wells of a single destination. | Span-8, Multichannel |
| Transfer From File | Use an input file to determine which wells to transfer. | Span-8 |
| Serial Dilution | Perform multiple dilutions of a sample on a single microplate. | Span-8 |
| Select Tips Serial Dilution | Perform multiple dilutions of a sample on a single microplate. | Multichannel, when using Select Tips |

---

## Common Pipetting JSON Keys

The following JSON keys are common to most pipetting steps, either on the step
itself for low-level pipetting or each operation in a step for high-level
pipetting. Refer to the individual step's reference documentation to confirm.

| Key | Type | Description |
|-----|------|-------------|
| `prototype` | string or expression | The string name of the pipetting technique to use |
| `autoSelectPrototype` | boolean | Whether the software picks the best technique automatically (not available on Fixed-8 pods) |
| `liquidType` | string or expression | The type of liquid being operated on. May be `Well Contents` for aspirates or mixes, `Tip Contents` for dispenses, or a named liquid type. |
| `overrideHeight` | boolean | Whether or not this step will use a custom height instead of the height configured in the technique |
| `customHeight` | boolean | True if `height` will be specified with an expression (only used when `overrideHeight` is `true`) |
| `height` | number or expression | Height offset value (only used when `overrideHeight` is `true`) |
| `heightFrom` | integer or expression | Reference point for the height (only used when `overrideHeight` is `true`) |

---

## Well Numbering

Pipetting steps address wells by a **1-based, row-major** well number: numbering runs
across the top row first, then down. For labware with `WellsX` columns and `WellsY` rows,
well `n` is at row `⌈n / WellsX⌉` and column `((n − 1) mod WellsX) + 1`.

For a standard 96-well plate (`WellsX = 12`, `WellsY = 8`):

- Row A = wells 1–12, row B = 13–24, …, row H = 85–96.

A multi-probe pod that spans a full **column** (Span-8, Fixed-8, a 96-head column) places
its first tip at `firstWell` and steps the remaining tips **down the column by `+WellsX`** —
i.e. wells `firstWell, firstWell+WellsX, firstWell+2·WellsX, …`. If that list runs past the
last row of the plate, the step fails at enqueue because the pod cannot place all its tips there.

**Consequence for looping over columns.** To visit plate columns `1..12` with a
column-spanning pod, the first-well expression is the **column index itself**:

```jsonc
// Loop variable Col = 1..12, dispensing a full 8-well column each pass:
"useWellExpression": true,
"firstWellExpression": "=Col"      // wells Col, Col+12, …, Col+84 on a 12-column plate
```

Do **not** write `"=1+(Col-1)*8"` — stepping by 8 assumes column-major numbering
(8 wells per column). On a row-major plate that lands the first tip mid-plate
(e.g. `Col=3` → well 17 = row B), leaving fewer than 8 rows below it, so the pod cannot fit
tip 8 and the step fails at enqueue. Multiply by a stride only when iterating rows — and the
correct row stride is `WellsX` (12 on a 96-well plate), not the pod's probe count.

### Reservoirs and multi-section troughs

Reservoir and trough labware is **not** addressed by the row-major well rule above. Its "wells"
are **sections**, and `WellsX` / `WellsY` do not apply. A section is addressed by its **1-based position in that labware type's section list** (valid range `1 … Sections.Count`), and a pipetting step's `selectionInfo`, `pattern`, and `sectionExpression` use those same 1-based section indices.

The section number is **not geometric.** When a labware type is built in the Labware Type
Editor, the author chooses the section order — "Right-Then-Down", "Down-Then-Right", or "No
Sorting" with each section's number assigned by hand — so section 1 is **not** guaranteed to be
the top-left section, and the numbering need not follow any physical progression.

Because the mapping is defined by the labware type, the only reliable source of truth is that
type's own section definition — inspect it in the Labware Type Editor, which pairs each
section's number with its physical rectangle, or read the corresponding local project file if you
have it. Do not assume a fixed top-to-bottom or left-to-right order. When you author a starting
section, the step fills its remaining tips from that section onward in list order.

---

## Tip Handling

Each pod defines its own Load Tips and Unload Tips steps, prefixed with the pod
type. Additionally, Select Tips groups define Select Tips Load Tips, Select
Tips Unload Tips, Advanced Load Tips, and Advanced Unload Tips.

Some low-level pipetting steps support a **Refresh Tips** option to load tips
prior to an operation, but generally speaking, tip handling occurs before and
after low-level pipetting steps.

High-level pipetting steps provide options for whether and how to change tips
between operations.

---

### Splitting a draw that will not fit

On a Transfer, Combine, or Transfer From File step, `splitVolume: true` breaks an oversized draw
into several **equally sized** cycles rather than one large one. The number of cycles is driven by
the smaller of two *usable* capacities — the tip's capacity after blowout and air gaps, and the
syringe or plunger capacity after air gaps and volume calibration — so neither raw tip capacity nor
`max.d` predicts it directly.

Splitting engages only when the aspirate serves a single destination, and repeat-by-volume mode
(`repeatsByVolume: true`) disables it as well. Raising `repeats` above 1 does not make the cycles
coarser — it turns splitting **off** and leaves the oversized draw to be attempted whole, on top of
grouping more destinations onto that one draw. Pair `splitVolume: true` with `repeats: "1"` and
`repeatsByVolume: false`. See [Transfer](step-documentation/11-Transfer.md) Cross-Field
Rule 17.
