# Multichannel Select Tips Advanced Load

> **Reference manual:** Chapter 20 documents these as *Select Tips* steps (the manual omits the *Multichannel* prefix); the JSON `stepType` keeps the `Multichannel Select Tips …` name.

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Select Tips Advanced Load"` |
| Category | Leaf (does not accept child steps) |
| Terminator | N/A |
| Compatible Hardware | Multichannel pod only |

## Behavior Summary

The Advanced Load Tips step loads tips from a specific row and column position
in a tip box, giving explicit control over which tips are picked up (unlike the
standard Load Tips which auto-selects). This step handles precise tip placement
when automatic pattern-based loading is insufficient. It also handles the case
where a tip box has been preconfigured with a pattern of tips.

The `row` and `column` values allow specifying where mandrel A1 is positioned
over the tip box. Whatever tips are underneath the head at that point are
loaded onto the pod.

This step must be placed inside a `"Multichannel Select Tips"` container. The
pod must not already have tips loaded.

Note that Multichannel Select Tips Advanced Load does not decrement the tip
reuse counter when tips are later returned to the box.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `tipType` | `string` | — | Yes | `""` | Tip **box** labware-class name, **bare and unprefixed** (e.g. `"BC230"`, `"BC190F"`, `"BC1025F"`). Same grammar as [Multichannel Select Tips Load](32-Multichannel-Select-Tips-Load.md) `tipType`; see [Catalog: Labware Classes](../biomek-file-formats/project-items/catalog-labware-classes.md#tip-boxes). |
| `tipsLocation` | `expression-capable string` | — | Yes | `""` | Deck position containing the tip box, tip box type, or tip box **instance** name. |
| `row` | `expression-capable string` | — | Yes | `"1"` | Row number in tip box to position mandrel A1 over (1-based; negative or zero values are allowed to offset the pod mandrel grid for partial-tip loading). |
| `column` | `expression-capable string` | — | Yes | `"1"` | Column number in tip box to position mandrel A1 over (1-based; negative or zero values are allowed to offset the pod mandrel grid for partial-tip loading). |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

## Cross-Field Validation Rules

1. Pod **must not** have tips loaded, or a load-with-tips error is raised.
2. `tipType` **must not** be empty.
3. Tip type must be compatible with the pod.
4. Position must contain a tip box with the correct tip type, or if a named tip box
   or tip box class is specified, it must match a tip box on deck.
5. `row` must evaluate to a valid integer; an empty or non-integer value raises a row-not-specified or invalid-row error.
6. `column` must evaluate to a valid integer *(same rules as `row`)*.
7. The `(row, column)` pair must resolve to at least one tip on the pod's mandrel grid; otherwise the step throws a loads-no-tips error.
8. This step **must** be inside a `"Multichannel Select Tips"` container; at method root it raises a container-required error.

## Structural Context

This step is a leaf inside a `"Multichannel Select Tips"` container.

## Canonical Examples

```json
{
  "stepType": "Multichannel Select Tips Advanced Load",
  "parameters": {
    "tipType": "BC230",
    "tipsLocation": "TL1",
    "row": "1",
    "column": "3"
  }
}
```

## Common Mistakes

- **Fictional or wrongly-shaped `tipType`**: Use a real, bare tip-box labware class name (e.g. `"BC230"`, `"BC190F"`, `"BC1025F"`, `"BC50F"`, `"BC90"`). Free-text descriptions like `"P250 Tips"` throw an unknown-labware-class error at runtime, and so does a `TipClasses\` / `LabwareClasses\` path — the prefixed form belongs on an Instrument Setup labware object, not here.
- **Out-of-range `row`/`column` that load nothing**: `(row, column)` may fall outside the tip box (including negative values, which offset the mandrel grid for partial-tip loading), but the resolved intersection must land on at least one tip — otherwise the step throws a loads-no-tips error.
- **Using this where standard Load Tips would do**: Advanced Load is for
  precise, single-position picks; for automatic full head loading, use
  `"Multichannel Select Tips Load"` instead.
