# Cleanup

| Property | Value |
|----------|-------|
| stepType | `"Cleanup"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7 |

> **Not available on i3 (Fixed-8).** The Cleanup step is i5/i7 only. On an i3, unload
> disposable tips before Finish with a `Fixed-8 Unload Tips` step, or rely on Finish's
> `cleanupPods: true` (the default), which handles tip disposal at method end.

---

## Behavior Summary

The Cleanup step directs the instrument to dispose of tips and optionally return tip boxes to their configured destinations. When executed, it iterates over both pods (Pod1 and Pod2) and removes any loaded tips. With no options selected (the default), the step only unloads tips to their configured unload location — tip boxes are left wherever they currently are. When `sendTipBoxes` is enabled, the step additionally sends all tip boxes that originated off-deck (and those configured to go to trash) to their final destinations. When `onDeckToHome` is also enabled, tip boxes that started on the deck are returned to their original on-deck positions as well. The Cleanup step resets the physical hardware state (tip and tip box positions) but does not reset software state — use it together with the Finish step to reset both hardware and software for repeated method runs. When used within a Select Tips group, the Cleanup step unloads tips to the configured unload location of the source tip box, or if that location is not available, discards tips to the trash.

---

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `sendTipBoxes` | `boolean` | — | Yes | `false` | Send tip boxes to their configured final destinations. When `false`, tip boxes are left wherever they are after tip disposal. When `true`, all tip boxes that originated off-deck and those configured to go to trash are sent to those final destinations. Omission raises a "could not find SendTipBoxes" system error. |
| `onDeckToHome` | `boolean` | — | Cond. | `false` | Return tip boxes that started on the deck back to their original on-deck locations. Only *meaningful* when `sendTipBoxes` is `true`. Must be present when `sendTipBoxes` is `true` — omitting it then raises a "could not find OnDeckToHome" system error. When `sendTipBoxes` is `false`, the key is never read and omission is harmless, though authoring `false` explicitly is recommended for clarity. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

---

## Enumerated / Constrained Values

Both parameters are simple boolean flags (`true` / `false`). There are no enumerated or string-constrained values for this step.

---

## Cross-Field Validation Rules

1. `sendTipBoxes` is always required at runtime — omitting it raises a "could not find SendTipBoxes" system error. `onDeckToHome` is required only when `sendTipBoxes` is `true` — omitting it then raises the analogous "could not find OnDeckToHome" system error; when `sendTipBoxes` is `false` the key is never read. The minimal safe authored form is `{ "sendTipBoxes": false, "onDeckToHome": false }` — authoring both keys explicitly is recommended for clarity.
2. `onDeckToHome` is **only meaningful when `sendTipBoxes` is `true`**. When `sendTipBoxes` is `false`, the value of `onDeckToHome` is ignored at runtime and omitting it is harmless. The UI enforces this by disabling the "Send boxes that started on deck back to their original locations" checkbox unless "Send tip boxes to their configured final destinations" is selected.
3. The two parameters are **independent of all other steps' parameters** — Cleanup does not reference or depend on any keys from Start, Finish, or other steps.

## Cross-Step Context: per-tip-box routing keys drive Cleanup's destinations

Cleanup does not carry per-tip-box routing itself; it reads the **labware properties**
that the Instrument Setup step attaches to each tip box and routes accordingly. Two
drop-downs on the Labware Properties dialog write those keys:

- **"When empty, send to"** → per-tip-box key `WhenDone`. Populated from
  the LabwareSetup UI with **`Home`**, **`Trash`** (`LOC_HOME`, `LOC_TRASH`) and any
  positions with the `Can Discard Labware` characteristic; the special sentinel
  **`<Home>`** appears as a menu item and stores literally that value.
- **"Unload Tips Into"** → per-tip-box keys `DiscardTips` (boolean) +
  `DiscardTipsLocation` (position; default `LOC_TRASH`).

When `sendTipBoxes: true`, Cleanup walks every tip box on the deck and routes each one
based on where it started: `onDeckToHome: false` routes only tip boxes that started
off-deck (leaving on-deck tip boxes in place), and `onDeckToHome: true` routes those
plus on-deck tip boxes back to their `WhenDone` destinations.

**Storage Setup + Cytomat interplay.** When a Storage Setup step supplies tip boxes
from a Cytomat and Cleanup runs with `sendTipBoxes: true`, those tip boxes are
returned via the storage device rather than the deck (tips retrieved from the
Cytomat in the setup case are returned via that same path in the
Cleanup+`onDeckToHome` case). This is a property of the deck's storage-aware
routing, not of a Cleanup-specific key.

---

## Structural Context

The Cleanup step is a **leaf** step with the following structural constraints:

- **Does not support `subSteps`** — never include a `subSteps` array on a Cleanup step.
- **May appear anywhere in the method body** — between the Start and Finish boundary steps, at the method root level or nested inside container steps (Group, Loop, If, etc.).
- **May appear inside a Select Tips group** — when nested inside a Select Tips container, the step unloads tips to the configured unload location of the source tip box rather than the default disposal behavior.
- **Multiple Cleanup steps are permitted** in a single method if needed (e.g., cleanup between different pipetting phases).

---

## Canonical Examples

### Minimal Cleanup (tips only — the most common case)

Disposes of all tips from both pods. Tip boxes are left in their current positions.

```json
{
  "stepType": "Cleanup",
  "parameters": {
    "sendTipBoxes": false,
    "onDeckToHome": false
  }
}
```

`sendTipBoxes` is required at enqueue — omitting it raises a "could not find SendTipBoxes" system error. `onDeckToHome` is required only when `sendTipBoxes: true`; omitting it when `sendTipBoxes: false` (as in this example) is harmless, though authoring it explicitly is recommended.

### Cleanup with tip boxes sent to configured destinations

Disposes of tips and sends tip boxes that originated off-deck (or are configured to go to trash) to their final destinations. On-deck tip boxes remain in place.

```json
{
  "stepType": "Cleanup",
  "parameters": {
    "sendTipBoxes": true,
    "onDeckToHome": false
  }
}
```

### Full Cleanup — tips and all tip boxes returned

Disposes of tips, sends off-deck tip boxes to their destinations, and returns on-deck tip boxes to their original starting positions. Use this when the method will run repeatedly and the deck must be fully reset between runs.

```json
{
  "stepType": "Cleanup",
  "parameters": {
    "sendTipBoxes": true,
    "onDeckToHome": true
  }
}
```

---

## Common Mistakes

- **Omitting `sendTipBoxes`** always raises a "could not find SendTipBoxes" system error. **Omitting `onDeckToHome` when `sendTipBoxes: true`** raises the analogous "could not find OnDeckToHome" system error. Omitting `onDeckToHome` when `sendTipBoxes: false` is harmless but not recommended. Author both keys explicitly for clarity.
- **Confusing Cleanup with Finish** — Cleanup resets *hardware* (tips, tip boxes). Software state (deck labware, globals) needs Finish.
- **Expecting Cleanup to clear the deck or park pods** — It only touches tips and tip boxes; deck labware state and pod parking are Finish's job.
- **Omitting Cleanup in repeated-run methods** — Without it, tips and tip boxes carry over from the previous run's ending positions.
