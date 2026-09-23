# Multichannel Select Tips

> **Reference manual:** Chapter 20 documents these as *Select Tips* steps (the manual omits the *Multichannel* prefix); the JSON `stepType` keeps the `Multichannel Select Tips …` name.

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Select Tips"` |
| Category | Free-form container |
| Terminator | `"Multichannel Select Tips End"` |
| Compatible Hardware | Multichannel pod only |

---

## Behavior Summary

The Select Tips container groups a sequence of select-tips operations (aspirate, dispense, mix, load, unload) that work with a specific multichannel pod using partial-tip patterns. At runtime, the container validates that the pod starts without tips, stores the pod and rearrange position globally for child steps to reference, enqueues all child steps in order, then cleans up. The pod **must have no tips loaded** by the time the `"Multichannel Select Tips End"` terminator executes (display caption `"End Using Select Tips"`, but the emitted stepType is `"Multichannel Select Tips End"`) — a Select Tips group that loads tips must also unload them before the terminator.

The rearrange position is an optional tip-box position used as temporary storage
when loading complex tip patterns that require multiple load/unload cycles.

Use Multichannel Select Tips when you need to load **less than a full head** of tips
on a multichannel pod. If you are using a full head of tips, do **not** use
Multichannel Select Tips; use Multichannel Load Tips and Multichannel Unload
Tips instead.

### Declaring the rearrange position's empty tip box

When a child Load actually needs the rearrange position, that position must hold a tip box which is
of the **same tip type** as the box being loaded from and is **empty** — no tips in any well. A box
that still holds tips fails with `The tip box on position "<position>" must be empty.`

A tip box's tip inventory is stored in the same per-well amounts array that `evalAmounts` seeds for
liquid labware: a well holding a tip has a positive amount, and an empty well is `0`. So an empty
tip box is declared in Instrument Setup exactly like a plate seeded to zero — set `volumeType` to
`"Known"` and give `evalAmounts` a full-length array of `0.0`:

```jsonc
{
  "_biomekType": "labware",
  "class": "LabwareClasses\\BC230",
  "volumeType": "Known",
  "evalAmounts": {
    "_biomekType": "comArray", "arraySubtype": "variant",
    "values": [0.0, 0.0, /* …96 total, one per well… */ 0.0]
  }
}
```

Place that object on a position carrying the multichannel tip-load characteristic (`tL*`), the same
as any other MC tip box, and name it in the container's `rearrangePos`. Note the array must be
full-length: a shortened array does **not** zero the remaining wells — see §`evalAmounts` /
`evalLiquids` in [Instrument Setup](03-Instrument-Setup.md).

Two related points that are easy to get backwards:

- **A declared tip box occupies the deck whether or not a step addresses it.** Obstruction follows
  declaration, not use, so an unused box on a neighboring position can still block the head. See
  [Reachability and Access](../biomek-file-formats/deck-layouts/reachability-and-access.md).
- **Clearing `rearrangePos` is not a way to avoid the rearrange machinery.** The value is only
  consulted when a load pattern genuinely needs staging; emptying it does not change which pattern
  you asked for, it just removes the position the load was going to use.

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | `string` | — | Yes | `""` | Multichannel pod name (e.g., `"Pod1"`). |
| `rearrangePos` | `string` | — | Conditional | `""` | Optional rearrange position for multi-step tip loading. |

> _`rearrangePos` becomes **conditionally required** only when a child that needs a temporary rearrange position runs — the child load step then raises a rearrange-position-not-specified error._

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

---

## Enumerated / Constrained Values

None. `pod` and `rearrangePos` are free-form strings validated at runtime.

---

## Cross-Field Validation Rules

1. Pod **must not** have tips loaded when this step begins, or a start-with-tips error is raised.
2. Pod **must have no tips loaded** when `"Multichannel Select Tips End"` executes, or an exit-with-tips error is raised. Include a Select Tips Unload / Advanced Unload child before the terminator.
3. A pipetting child step (Aspirate, Dispense, Mix, Serial Dilution) that runs before tips are loaded raises a pod-no-tips error; Unload / Advanced Unload raise a similar unload-no-tips error for the same condition.
4. Any Select Tips child step placed outside a `"Multichannel Select Tips"` container raises a container-required error.
5. If `rearrangePos` is specified, it must be a valid deck position that can load tips.

---

## Structural Context

This step is a **free-form container**. Child steps are placed in `subSteps`, terminated by `"Multichannel Select Tips End"`.

Valid child step types: [`"Multichannel Select Tips Aspirate"`](29-Multichannel-Select-Tips-Aspirate.md), [`"Multichannel Select Tips Dispense"`](30-Multichannel-Select-Tips-Dispense.md), [`"Multichannel Select Tips Mix"`](31-Multichannel-Select-Tips-Mix.md), [`"Multichannel Select Tips Load"`](32-Multichannel-Select-Tips-Load.md), [`"Multichannel Select Tips Unload"`](33-Multichannel-Select-Tips-Unload.md), [`"Multichannel Select Tips Advanced Load"`](34-Multichannel-Select-Tips-Advanced-Load.md), [`"Multichannel Select Tips Advanced Unload"`](35-Multichannel-Select-Tips-Advanced-Unload.md), [`"Multichannel Select Tips Serial Dilution"`](28-Multichannel-Select-Tips-Serial-Dilution.md) (all from the Select Tips palette).

```jsonc
{
  "stepType": "Multichannel Select Tips",
  "parameters": { ... },
  "subSteps": [
    { "stepType": "Multichannel Select Tips Load", ... },
    { "stepType": "Multichannel Select Tips Aspirate", ... },
    { "stepType": "Multichannel Select Tips Dispense", ... },
    { "stepType": "Multichannel Select Tips Unload", ... },
    { "stepType": "Multichannel Select Tips End", "parameters": { } }
  ]
}
```

---

## Canonical Examples

Basic select tips workflow:

```json
{
  "stepType": "Multichannel Select Tips",
  "parameters": {
    "pod": "Pod1",
    "rearrangePos": ""
  },
  "subSteps": [
    {
      "stepType": "Multichannel Select Tips Load",
      "parameters": {
        "tipType": "BC190F",
        "tipsLocation": "TL1",
        "pattern": "Columns",
        "columns": "1"
      }
    },
    {
      "stepType": "Multichannel Select Tips Aspirate",
      "parameters": {
        "location": "P3",
        "labwareClass": "BCFlat96",
        "volume": "100",
        "columnOffset": "1",
        "rowOffset": "1",
        "liquidtype": "Well Contents",
        "autoSelectPrototype": true
      }
    },
    {
      "stepType": "Multichannel Select Tips Dispense",
      "parameters": {
        "location": "P4",
        "labwareClass": "BCFlat96",
        "volume": "100",
        "columnOffset": "1",
        "rowOffset": "1",
        "liquidtype": "Tip Contents",
        "autoSelectPrototype": true
      }
    },
    {
      "stepType": "Multichannel Select Tips Unload",
      "parameters": {
        "unloadPosition": "TL1"
      }
    },
    { "stepType": "Multichannel Select Tips End", "parameters": { } }
  ]
}
```

---

## Common Mistakes

- **Tips on the pod at entry or exit**: Select Tips brackets a *tips-off → tips-off* region. Pre-existing tips at start raise a start-with-tips error (see rule 1); tips loaded inside and not unloaded before the terminator raise an exit-with-tips error (see rule 2).
- **Omitting `rearrangePos` when a child Load needs it**: the container accepts an empty value, but Load raises a rearrange-position-not-specified error at runtime when the pattern requires a temporary tip-box position.
- **Using Select-Tips child steps outside this container**: any of the Select-Tips-family leaves raises a container-required error at method root (see rule 4).

---

## Related Guides

- **[Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md)** — technique auto-selection applies to Select-Tips-family child pipetting steps.
