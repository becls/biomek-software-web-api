# Multichannel Select Tips Load

> **Reference manual:** Chapter 20 documents these as *Select Tips* steps (the manual omits the *Multichannel* prefix); the JSON `stepType` keeps the `Multichannel Select Tips …` name.

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Select Tips Load"` |
| Category | Leaf (does not accept child steps) |
| Terminator | N/A |
| Compatible Hardware | Multichannel pod only |

## Behavior Summary

The Select Tips Load step picks up tips from a tip box using the multichannel pod, loading a specific pattern of tips (single tip, specific rows, or specific columns). It supports a primary tip location and an optional backup location. When loading rows or columns, if tips are not available contiguously, the step can use the container's rearrange position to perform multi-step loading (load partial, unload to rearrange, load more).

This step must be placed inside a `"Multichannel Select Tips"` container. The pod must not already have tips loaded.

When the requested `"Rows"`/`"Columns"` pattern cannot be satisfied by a single pick from the available tips (e.g. the requested rows/columns aren't the topmost/rightmost tips in the box), the parent `"Multichannel Select Tips"` container must have `rearrangePos` set to a deck position holding an empty tip box of the same type — the child Load rearranges through that position. Without it, enqueue throws a rearrange-position-not-specified error; with a position whose box still holds tips, it throws `The tip box on position "<position>" must be empty.` For the JSON that declares an empty tip box — which is not the same shape as omitting the box — see [§Declaring the rearrange position's empty tip box](26-Multichannel-Select-Tips.md#declaring-the-rearrange-positions-empty-tip-box) in [Multichannel Select Tips](26-Multichannel-Select-Tips.md).

Budget the rearrange box in **addition** to the boxes you draw tips from: it
occupies a tip-load position for the whole method and supplies no tips.

It is recommended to name tip boxes used by Multichannel Select Tips steps and
reference them by name, to avoid sharing boxes between regular Multichannel Load
Tips/Unload Tips steps and Select Tips steps. Select Tips steps do **not**
decrement the tip reuse counter, so boxes used by select tips will appear usable
to Multichannel Load Tips steps after the Select Tips group even though they may
be dirty.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `tipType` | `string` | — | Yes | `""` | Tip **box** labware-class name, **bare and unprefixed** (e.g. `"BC190F"`, `"BC50F"`). Must match a labware class installed on the instrument, and its `WellsX`/`WellsY` must match the pod's mandrel grid. Must not be empty. See [Catalog: Labware Classes](../biomek-file-formats/project-items/catalog-labware-classes.md#tip-boxes). |
| `tipsLocation` | `string` | — | Yes | `""` | Deck position containing the tip box (e.g. `"P1"`) or labware **instance** name (as specified in Instrument Setup step). This is not a labware-class name — the class is `tipType`. Update it only when the tip box moves to a different position, not when its class changes; contrast Multichannel/Span-8 Load Tips, where `tips` names a tip *class*. See [Labware & Position Resolution](../concept-guides/02-labware-and-position-resolution.md#4-labware-must-be-declared-before-it-is-used) §4-5. |
| `backupTipsLocation` | `string` | — | No | `""` | Backup tip box position (used if primary is empty). |
| `pattern` | `string` | — | No | `"Single"` | Loading pattern. See enumeration below. |
| `rows` | `expression-capable string` | — | Conditional | `"1"` | Comma-separated row numbers (1-based). Used when `pattern` is `"Rows"`. |
| `columns` | `expression-capable string` | — | Conditional | `"1"` | Comma-separated column numbers (1-based). Used when `pattern` is `"Columns"`. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

## Enumerated / Constrained Values

### pattern Values

| Value | Meaning |
|-------|---------|
| `"Single"` | Load exactly one tip |
| `"Rows"` | Load all tips in specified row(s) |
| `"Columns"` | Load all tips in specified column(s) |

Note that the pattern indicates **where on the pod the tips will be loaded**,
not the location in the box to load. For example, selecting the right-most
column of a tip box will load those tips onto the right-most column of mandrels
on the pod.

## Cross-Field Validation Rules

1. Pod **must not** have tips loaded, or a load-with-tips error is raised.
2. `tipType` **must not** be empty, or a tips-none error is raised.
3. Tip type must be compatible with the pod.
4. Primary position must contain a tip box with the specified tip type.
5. If `backupTipsLocation` is specified, it must also contain the correct tip type.
6. For `"Rows"` pattern: rows must be valid integers; tip box must have complete rows available.
7. For `"Columns"` pattern: columns must be valid integers; tip box must have complete columns available.
8. Partial rows/columns are not allowed *(throws "incomplete row/column")*.

## Structural Context

This step is a leaf inside a `"Multichannel Select Tips"` container.

## Canonical Examples

Load a full column of tips:

```json
{
  "stepType": "Multichannel Select Tips Load",
  "parameters": {
    "tipType": "BC190F",
    "tipsLocation": "P1",
    "pattern": "Columns",
    "columns": "1"
  }
}
```

Load rows 1 and 2:

```json
{
  "stepType": "Multichannel Select Tips Load",
  "parameters": {
    "tipType": "BC50F",
    "tipsLocation": "P1",
    "backupTipsLocation": "P2",
    "pattern": "Rows",
    "rows": "1,2"
  }
}
```

Load a single tip:

```json
{
  "stepType": "Multichannel Select Tips Load",
  "parameters": {
    "tipType": "BC190F",
    "tipsLocation": "P1",
    "pattern": "Single"
  }
}
```

## Common Mistakes

- **Partial row/column anywhere in the box**: The step scans the tip box (right-to-left for `"Columns"`, bottom-to-top for `"Rows"`) looking for full columns/rows. If it encounters any column (or row) with some tips present and some missing — even one that wasn't specifically requested — enqueue throws. Fully empty columns/rows are fine and are skipped. The step will not silently pick up a partial set.
- **`backupTipsLocation` misunderstood**: It's only consulted when the primary box is empty for the requested tips; it is not a second-pattern load.
- **Missing `rows`/`columns` for the chosen pattern**: `pattern: "Rows"` requires `rows`; `pattern: "Columns"` requires `columns`. Set the value to match your layout; the default `"1"` loads only the first row or column.

## The `Error cleaning tips for transfer from <position>:` message

This message does **not** come from any Select Tips step. It is emitted by a **Transfer** step
running on a Multichannel or Fixed-8 pod, which wraps its internal tip-provisioning routine in this
prefix. `<position>` is the **source** deck position of the transfer that was about to run — it is
not the tip box, and not the failing location. The real diagnosis is the message text **after the
colon**, which is whatever the wrapped work raised.

**It is not a reliable indicator of tip reuse.** Despite the "cleaning" wording, the routine is the
transfer's general tip-provisioning path, and it is invoked for the **first transfer of the step
unconditionally** — before any between-source or between-destination consideration applies. Turning
`changeTipsBetweenDests` off, turning `changeTipsBetweenSources` off, or turning `useJIT` off will
not suppress it.

What the wrapped routine does, in order:

1. Reads the pod's tips — the whole-head `Tips` on a Multichannel pod, or the validated per-probe
   set on a Fixed-8. A pod that exposes neither raises `No tips`.
2. On the first (checking) call it can **exit doing nothing**: when the pod already carries tips
   whose class name, labware name or origin position matches the transfer's tip location — or when
   `useCurrentTips` is set — and those tips are unused (or clean, if active wash is on).
3. Otherwise it either **washes** the tips (active wash, and the tips are already the right type),
   or **enqueues a tip load**: a Multichannel Load Tips step on a Multichannel pod, or an
   Unload-then-Load pair on a Fixed-8. The tip location it passes is the transfer's own
   `tipLocation`.

So the whole failure surface of tip loading, tip unloading and washing can surface behind this one
prefix. Bodies you are most likely to see:

| Message after the colon | Meaning |
|---|---|
| `Tips not specified.` | The transfer has no tip location and is not reusing current tips. |
| `Cannot use current tips; no tips loaded.` | `useCurrentTips` is set but the pod is empty. |
| `No tips` | The pod exposes no tip state — usually the wrong pod type for the step. |
| `Unable to find and retrieve tips of type <value> for <pod>.` | The tip load failed; the appended per-box skip reasons say why. See [Multichannel Load Tips](22-Multichannel-Load-Tips.md). |
| `The fixed-8 pod does not support washing tips.` | Active wash was requested on a Fixed-8 pod. |
| a tip-discard or unload error | The Fixed-8 unload half failed — see [Multichannel Unload Tips](23-Multichannel-Unload-Tips.md) for how a discard location is resolved. |
