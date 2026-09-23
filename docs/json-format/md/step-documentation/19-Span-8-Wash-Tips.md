# Span-8 Wash Tips

| Property | Value |
|----------|-------|
| stepType | `"Span-8 Wash Tips"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5, i7 (Span-8 pod required) |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Universal base keys (`caption`, `defaultCaption`, `dynamic?`, `disabled`, …)
> are documented once in that file and are **not** repeated here.

## Behavior Summary

The Span-8 Wash Tips step rinses tips or the disposable tip mandrels on a Span-8 pod at a
wash station. It runs in one of two modes.

In **passive** mode (`activeWash: false`,
the default) the step moves the selected probes to a passive Span-8 wash station
(deck characteristic `"VWash Station"`), dispenses `toWaste` mL to the waste channel,
then dispenses `amount` mL of system liquid while over the wells to rinse; both volumes
are entered in **mL and multiplied by 1000 to µL** internally. In **active** mode
(`activeWash: true`) the step instead enqueues a Span-8 mix at an active wash station
(characteristic `"Wash Station"`), performing `activeWashCycles` cycles of `activeVolume`
of the `activeLiquidType` solvent using the selected (or auto-selected) technique.

At enqueue the step resolves the pod (which must be a Span-8 pod), reads the probe
selection, eliminates probes whose tips are clean when `dirtyTipsOnly` is set (or that
have no content when `dispenseTipContentsOnly` is set), verifies all remaining tips are
the same type, and finds the wash station — using `where` if given, otherwise the nearest
reachable station of the correct type. After washing it removes tip contents, marks the tips
clean (unless `dispenseTipContentsOnly` is set, which only empties them), and records
`"Wash"` as the pod's last operation. The step is a **leaf**: it expands into lower-level
pod motions (and, for active wash, a Span-8 mix) at runtime, but is authored with no
`subSteps`.

> **Dispensing waste through the wash station.** Emptying tip contents into the wash station's
> waste channel (the `toWaste` volume, or `dispenseTipContentsOnly`) is a common way to discard
> liquid during a run. It is generally paired with a rinse that dilutes and clears what was sent
> down — `amount` mL of system fluid over the wells on a passive wash, or an active-wash cycle.
> Whether a **particular** liquid is appropriate to send through the wash station depends on the
> liquid and your instrument's plumbing, and is outside the scope of these docs. If you are not sure
> a liquid is safe to dispense through the wash station, check with Beckman Coulter before relying
> on it.

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | string (expression) | — | Yes | `""` | Name of the Span-8 pod (e.g. `"Pod1"`). Must resolve to a Span-8 pod. |
| `where` | string (expression) | — | No | `""` (auto-find) | Deck position of the wash station. Blank ⇒ the step finds the nearest reachable station — a `"VWash Station"` for passive wash or a `"Wash Station"` (matching `activeLiquidType`) for active wash. |
| `activeWash` | boolean | — | No | `false` | `false` = passive wash (dispense to waste + rinse in wells); `true` = active wash (mix `activeVolume` × `activeWashCycles` at an active wash station). Selects which volume/technique keys apply. |
| `amount` | string (expression) | mL | Conditional | `"1"` | **Passive** rinse volume dispensed while over the wells (converted ×1000 to µL). Used only when `activeWash` is `false`. A trailing `%` is allowed when tips are loaded; entering the volume as a percent specifies the volume of wash fluid as a percent of the maximum volume of fluid contained in the tips in previous steps. For example, if the maximum volume of fluid transferred is 50 μL, and the % is set for 110%, the Wash step washes the tips with 55 μL of solution. However, if no fluid has been transferred and the volume is entered as a percent, no action is taken and tips are not washed. |
| `toWaste` | string (expression) | mL | No | `"2"` | Volume dispensed to the waste channel before the in-well rinse (converted ×1000 to µL). The StepUI writes `"2"`; if omitted, enqueue falls back to `1` mL.|
| `delayTime` | string (expression) | ms | No | `"300"` | Delay after each dispense cycle. Must be a non-negative number. The StepUI writes `"300"`; if omitted, enqueue falls back to `0` ms. |
| `dirtyTipsOnly` | boolean | — | No | `true` | When `true`, only probes whose tips are marked *dirty* are washed; clean tips are skipped. Defaults to `true` when omitted.
| `dispenseTipContentsOnly` | boolean | — | No | `false` | **Passive** only. When `true`, the step only empties the remaining tip contents to waste (no fresh rinse); probes with empty tips are skipped.|
| `activeVolume` | string (expression) | µL or % | Conditional | `""` | **Active** wash volume per cycle. Used only when `activeWash` is `true`. A trailing `%` is allowed when tips are loaded; entering the volume as a percent specifies the volume of wash fluid as a percent of the maximum volume of fluid contained in the tips in previous steps. For example, if the maximum volume of fluid transferred is 50 μL, and the % is set for 110%, the Wash step washes the tips with 55 μL of solution. However, if no fluid has been transferred and the volume is entered as a percent, no action is taken and tips are not washed. |
| `activeWashCycles` | string (expression) | — | Conditional | `""` | **Active** wash cycle count — passed to the enqueued mix as `C__MixCount`. Used only when `activeWash` is `true`.
| `activeLiquidType` | string | — | Conditional | `""` | **Active** wash solvent liquid type — it becomes the `liquidType` of the mix the step enqueues at the wash station. **Required whenever `activeWash` is `true`**: the key must be present (it is read with no fallback), so omitting it fails the step even when `where` names the station explicitly. Additionally, when `where` is blank it selects the wash station by solvent, and *that* lookup alone tolerates omission by falling back to `"Water"`. Ignored when `activeWash` is `false`. |
| `operation` | string | — | No | `"Mix"` | Technique-selector operation tag. Always `"Mix"` for this step. |
| `autoSelectPrototype` | boolean | — | Conditional | `false` | Active-wash technique auto-selection. Only relevant when `activeWash` is `true`. Omission defaults to `false` (named-technique mode), which then requires a non-empty `prototype`; set it explicitly to `true` to opt into auto-selection. See [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md). |
| `prototype` | string | — | Conditional | `""` | Active-wash technique name (used when `autoSelectPrototype` is `false`). |
| `customPrototype` | object (technique) | — | No | *(absent)* | Inline custom active-wash technique object. Present **only** when a custom technique was defined in the editor — absent from normal exports. |
| `useProbes` | comArray (boolean[8]) | — | No | *(all fixed probes)* | Per-probe selection (indices 0–7). Serialized as `{"_biomekType": "comArray", "arraySubtype": "boolean", "values": [...]}`. When omitted, enqueue falls back to the pod's fixed-probe array. |
| `mandrelExpression` | string | — | No | `""` | Expression-based probe selection. Used when `useExpression` is `true`. |
| `useExpression` | boolean | — | No | `false` | When `true`, probe selection comes from `mandrelExpression` instead of `useProbes`. |
| `openPosition` | boolean | — | No | `true` | When `true`, the deck position is opened before and closed after washing (passive path). Read by enqueue but **not written by the editor**; safe to omit. |

### Inert / inactive keys

| Key | Classification | Notes |
|-----|----------------|-------|
| `useSpeedPump` | **Inactive — not read at runtime** | Not referenced by the i-Series Span-8 step; left over from previous Biomek FX/NX generation. This key has no effect on this step. Do not rely on it. |
| `caption` | base | See [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md). Node-level `disabled` toggles whether the step runs. |

There is no `items` array and no sub-object array — the Item-Level Keys section does not apply.

## Enumerated / Constrained Values

### activeWash
| Value | Meaning |
|-------|---------|
| `false` | Passive wash: dispense `toWaste` mL to waste, then rinse `amount` mL in the wells (fixed tips, empty disposable tips, or disposable tip mandrel dispense only). |
| `true` | Active wash: mix `activeWashCycles` cycles of `activeVolume` of `activeLiquidType` at an active wash station. |

### operation
| Value | Meaning |
|-------|---------|
| `"Mix"` | Only value. The wash is executed as a mix operation for technique selection. |

### amount / activeVolume percent form
| Form | Meaning |
|------|---------|
| plain number (e.g. `"2"`) | Absolute volume (mL for passive `amount`/`toWaste`; µL for active `activeVolume`). |
| trailing `%` (e.g. `"110%"`) | Percent of the tip's `MaxVolumeUsed`. Requires tips to be present (else validation fails). |

## Cross-Field Validation Rules

Each rule describes the enqueue-time behavior it enforces.

1. **Pod type** — `pod` must be a Span-8 pod, else the step raises a wrong-pod-type error.
2. **Wash volume parses to a number** — a non-numeric `amount`/`activeVolume` raises a "bad wash volume" error.
3. **Wash volume non-negative** — a negative value raises a "must be positive" error; an overflow raises a "too large" error.
4. **Percent volume needs tips** — a `%` volume with no tips loaded raises a "cannot be specified as percent" error.
5. **Tip present on selected probe** (mixed tip state) — a selected probe with no tip raises a per-probe error.
6. **Uniform tip type** — all selected probes with tips must share a type, else a mixed-type error fires.
7. **Disposable tips + passive wash** — disposable tips cannot be passively washed unless `dispenseTipContentsOnly` is set.
8. **Wash station resolvable** — if `where` is blank and none is found, or a named `where` is absent, or a named station is the wrong kind, each fails with a distinct error.
9. **Dispense-tip-contents needs tips** — `dispenseTipContentsOnly` with no tips raises a "no tips loaded" error.
10. **Delay time** — non-numeric, negative, and overflow values each fire a distinct error.
11. **Probe selection non-empty** — if no probes remain selected after the dirty/empty filters the step simply does nothing; it is **not** an error.

## Structural Context

This step is a **leaf** — it does not support authored `subSteps`. (At runtime it enqueues
pod motions and, for active wash, a Span-8 mix; those are engine-generated, not part of the
authored method tree.)

## Canonical Examples

> The examples use `"pod": "Pod1"`. The Span-8 pod is `Pod1` on a standalone Span-8 instrument
> (i5-Span, i7-Span) and `Pod2` (the second/rightmost pod) on an i7 Hybrid. `Pod1`
> is valid on a standalone Span-8. Use whichever pod your
> instrument defines.

### Example 1: Passive wash, all probes, auto-find station

```json
{
  "stepType": "Span-8 Wash Tips",
  "parameters": {
    "activeLiquidType": "Water",
    "activeVolume": "",
    "activeWash": false,
    "activeWashCycles": "",
    "amount": "1",
    "autoSelectPrototype": true,
    "delayTime": "300",
    "dirtyTipsOnly": true,
    "dispenseTipContentsOnly": false,
    "mandrelExpression": "",
    "operation": "Mix",
    "pod": "Pod1",
    "prototype": "",
    "toWaste": "2",
    "useExpression": false,
    "useProbes": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true, true, true, true, true, true, true, true]
    },
    "where": ""
  }
}
```

### Example 2: Active wash, 3 cycles of 150 µL Ethanol, named station, subset of probes

```json
{
  "stepType": "Span-8 Wash Tips",
  "parameters": {
    "activeLiquidType": "Ethanol",
    "activeVolume": "150",
    "activeWash": true,
    "activeWashCycles": "3",
    "amount": "1",
    "autoSelectPrototype": false,
    "delayTime": "300",
    "dirtyTipsOnly": true,
    "dispenseTipContentsOnly": false,
    "mandrelExpression": "",
    "operation": "Mix",
    "pod": "Pod1",
    "prototype": "S8 Active Wash",
    "toWaste": "2",
    "useExpression": false,
    "useProbes": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true, true, true, true, false, false, false, false]
    },
    "where": "AW1"
  }
}
```

## Common Mistakes

- **Treating `toWaste` / `amount` as booleans** — both are volumes in **mL** (strings/expressions), not "send to waste?" flags. The unit split: passive `amount`/`toWaste` are mL (×1000 internally), but active `activeVolume` is µL. `delayTime`, sitting alongside them, is neither — it is milliseconds. Three units in one parameter block, so read the Units column rather than pattern-matching the neighboring keys; a value copied from `amount` into `activeVolume` is a 1000× volume error.
- **Mixing passive and active volume keys** — `activeWash: false` ignores `activeVolume`/`activeWashCycles`/`activeLiquidType`; `activeWash: true` ignores `amount`/`toWaste`. Leave the unused side as empty strings (as the StepUI does) rather than co-authoring both.
- **Passive-washing disposable tips without `dispenseTipContentsOnly`** — passive station rejects disposables outright; the tip-contents flag is the only escape hatch.
- **Providing `useProbes` as a plain JSON array** — must be the `{"_biomekType": "comArray", ...}` wrapper object.
- **Pointing active wash at a non-wash-station position** — an active wash needs an **active** wash station ALP (a station plumbed for the solvent) on the deck, and `where` names it (leave it empty, as above, to let the engine locate it). The stock i5/i7 Span-8 sample decks include only a *passive* station (`W1`) and no active one, so a real active wash requires a site-specific active-wash position your instrument actually has — not a plate slot.
- **Authoring `useSpeedPump`** — inactive; ignored. Do not add it.

## Related Guides

- **[Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md)** — how `autoSelectPrototype`/`prototype`/`customPrototype` resolve the active-wash technique.
- **[Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)** — base keys, casing, `disabled`.
