# Concept Guide: Probe & Mandrel Selection

Which probes/mandrels of a pod are active for a step are defined in Span-8
`spacing`, `useProbes`, and `mandrelExpression`; Fixed-8 `numberOfTips`; and the
Multichannel Select Tips container family for partial-head Multichannel work.

---

## 1. Well spacing (`spacing`)

`spacing` applies to **Span-8 only.** Fixed-8 spacing is implicit (the arm pitch
is fixed) and is not a JSON-settable key.

`spacing` is the interval between the wells that adjacent probes address: `1` = every well
(adjacent rows), `2` = every other well, and so on. Omit `spacing` or set it to `"1"` for the
standard interval (every well on a 96-well plate). Set it to `"2"` on a 384-well plate (a
Span-8 arm is pitched for 9 mm rows; 384-well plates have 4.5 mm rows, so the minimum
reachable interval is 2). A spacing smaller than the labware allows (e.g. `1` on
a 384) is rejected.

## 2. Probe / mandrel selection

Not every probe has to be active. A pod's active probes are chosen one of two ways, selected
by a flag (see [Concept Guide: Expressions](01-expressions.md) for the flag/expression pairing):

- **Static** — `useProbes`, an **8-element boolean comArray**, one entry per probe (1–8),
  `true` = active. At least one must be `true`. This is the **Span-8** form (Fixed-8 uses a
  count via `numberOfTips` instead — see below).

  ```jsonc
  "useProbes": {
    "_biomekType": "comArray", "arraySubtype": "boolean",
    "values": [true, true, false, false, false, false, false, false]  // probes 1–2 only
  }
  ```

- **Expression** — set `useExpression: true` and give `mandrelExpression` an expression that
  resolves to the probe list at run time, e.g. `"=[1,2,3,4,5,6,7,8]"`.

For Span-8 note the physical constraint from [Introduction to Biomek and Liquid Handling Method Writing](../Introduction-to-Biomek-Method-JSON.md#pod-types) §"Pod Types": tip type is
configured per group of four probes (1–4, 5–8), and **all selected probes must share tip
type and syringe size** — you cannot co-select a fixed-tip probe and a disposable-tip probe.

For **Fixed-8**, the count of active tips is given as `numberOfTips` (a string,
`"1"`–`"8"`, expression-capable). Fixed-8 does not have a per-probe boolean mask like
Span-8's `useProbes`. For simple pipetting steps (aspirate/dispense/mix), the
Fixed-8 pod must first load tips with a Load Tips step, which is where
`numberOfTips` is set. The loaded tips are then used for the pipetting step.

For the Transfer step, `numberOfTips` is set directly on the Transfer step's dictionary.

## 3. Span-8 probe-selection keys by step type

Span-8 probe-selection keys are **not interchangeable across step types**. Carrying a key onto a step that does not read it is wrong -- it is silently ignored, and the intended probe selection does not take effect. Check the scope note on each key below.

### Simple Span-8 steps (Aspirate, Dispense, Load Tips, Unload Tips, Wash Tips)

| Key | Type | Role |
|-----|------|------|
| `useProbes` | boolean comArray[8] | Static probe mask (one entry per probe, at least one `true`) |
| `useExpression` | boolean | **Authoritative flag.** When `false`, the engine reads `useProbes`; when `true`, it reads `mandrelExpression` instead. |
| `mandrelExpression` | string | Run-time expression resolving to the probe list (only read when `useExpression` is `true`) |

These three keys are the complete probe-selection interface for simple Span-8 steps.

### Tip-mode keys: Transfer, Combine, Span-8 Transfer From File, Span-8 Serial Dilution

These four steps carry the three keys above **plus** a tip-mode group. The tip-mode keys are a
different way of choosing probes: instead of naming probes, they say "use whichever probes carry
this kind of tip", and when either is `true` the `useProbes` mask is not consulted.

| Key | Type | Role |
|-----|------|------|
| `useFixedTips` | boolean | Use the pod's fixed-tip probes. Read on `Transfer`, `Combine`, `Span-8 Transfer From File`, and `Span-8 Serial Dilution`. Not read on the simple Span-8 steps. |
| `useDisposableTips` | boolean | Use the pod's disposable-tip probes. Same four steps as `useFixedTips`. |
| `useMandrelSelection` | boolean | **Inert (UI-state)** -- written alongside the two above, but the engine does not read this key on any step |

Set at most one of `useFixedTips` / `useDisposableTips`. With both `false`, probe selection falls
back to the expression/mask keys.

### Span-8 Serial Dilution only

Serial Dilution additionally carries two flags that exist on no other step:

| Key | Type | Role |
|-----|------|------|
| `useMandrelExpression` | boolean | **Authoritative probe-source flag** (replaces the role of `useExpression` for probes on this step) |
| `useSectionExpression` | boolean | **Authoritative well-source flag** for section/well expressions |
| `useExpression` | boolean | Non-authoritative on Serial Dilution -- acts as UI scratch state only; the engine reads `useMandrelExpression` and `useSectionExpression` instead |

**Key difference:** on Serial Dilution, `useExpression` does **not** control whether `mandrelExpression` is read -- `useMandrelExpression` does. On simple steps, `useExpression` is the authoritative flag.

> `useMandrelSelection` is inert UI-state on every step that carries it. The engine never reads it. Do not author it expecting runtime behavior.

---

## 4. Partial-head Multichannel: the Select Tips family

A Multichannel head normally loads/pipettes a full grid. To use only *some* mandrels of the
head, wrap the operations in a **Multichannel Select Tips** container (terminator:
`"Multichannel Select Tips End"`) and use its child steps (`… Select Tips Load`,
`… Aspirate`, `… Dispense`, `… Unload`, etc.). Rules that matter:

- The pod must have **no tips** when the container begins, and **no tips** when the
  terminator runs — a Select Tips group that loads tips must also unload them before the
  `End`.
- Complex partial patterns that need multiple load/unload cycles use an optional
  `rearrangePos` (a tip-box position for temporary storage); it becomes required only when a
  child actually needs it.

This container is the sanctioned way to do partial-mandrel work on a Multichannel pod.
Span-8 selects probes directly with `useProbes`; Fixed-8 selects a count with
`numberOfTips`.

## 5. Quick checklist

- `spacing` (Span-8 only) = interval between probes: `1` = every well (96-well plate),
  `2` = every other well (384-well plate).
- Static probe mask = 8-boolean `useProbes` comArray (≥1 true); or `useExpression` +
  `mandrelExpression`.
- Fixed-8: active-tip count is `numberOfTips` (`"1"`–`"8"`), no per-probe mask.
- Partial Multichannel work goes inside a Multichannel Select Tips container, and must
  unload all tips before its terminator.
- Well numbering / `firstWell` math — see [Introduction to Pipetting Steps](../Introduction-to-Pipetting-Steps.md#well-numbering) §"Well Numbering".
