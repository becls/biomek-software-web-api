# Multichannel Wash Tips

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Wash Tips"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | i5 or i7 with a Multichannel pod |

> Base keys + casing: see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md).
> Universal base keys (`caption`, `dynamic?`, `disabled`, …) are **not** re-documented here.

---

## Behavior Summary

The Multichannel Wash Tips step washes the tips currently on a multichannel head
by mixing them in a wash-station reservoir. It validates the pod, tips, and
wash station, then performs a mix step in the wash station.

Tips must be empty before washing. Dispense the contents first using a
Multichannel Dispense step, then wash.

Note that Multichannel Aspirate, Multichannel Dispense, Multichannel Mix, and
Multichannel Wash Tips share much of their implementation. As a result, some
parameters are listed here that are only used by those other steps. See each
step's documentation for details.

---

## Parameters Reference Table

### Step-Level Keys

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `pod` | string | — | Yes | `""` | Name of the multichannel pod to use (e.g., `"Pod1"`). Must resolve to a multichannel pod. The MC pod is `"Pod1"` on a standalone Multichannel instrument (i5-MC, i7-MC) and on an i7 Hybrid; it is `"Pod2"` only when the MC is the right-hand pod of a dual-pod i7. Never author `"MC"`/`"MPod"`. See [Introduction to Biomek Method JSON](../Introduction-to-Biomek-Method-JSON.md#instrument-variants) §"Instrument Variants" / "Pod Types". |
| `location` | string | — | No | `""` | Deck position of the wash station. If empty, the step auto-selects the first reachable wash station whose solvent matches `liquidtype`. If set, it must be a wash station stocked with that solvent. Supports `=` expression strings (e.g., `"=xTips_WashPosition"`). |
| `liquidtype` | string | — | Yes (key must be present) | `""` | Solvent/liquid type used for the wash. The **key must be present**, and there must be a wash station on the deck that has that liquid type assigned. **Casing note**: the exported key is all-lowercase `liquidtype` (not `liquidType`) for this step — author it that way to match the editor's output. Expression-capable. |
| `volume` | string (number, percent, or expression) | µL or % | Yes | `"0"` | Wash volume. A plain number string (e.g., `"170"`) is µL; a trailing-`%` string (e.g., `"110%"`) is a percentage of the tips' max-volume-used, clamped to tip capacity; a leading-`=` string (e.g., `"=TipWashVol"`) is evaluated as an expression. Empty string raises a volume-not-specified error; a computed value `≤ 0` makes the step a no-op. Canonical export form is always a string; a plain JSON number imports correctly but does not match the editor's output. |
| `mixCount` | string (integer or expression) | — | Yes | `"1"` | Number of wash (mix) repetitions within the wash solution. Canonical export form is a string (e.g., `"2"`, `"4"`, or an expression like `"=TipWashCycles"`). Must evaluate to a positive integer. |
| `autoSelectPrototype` | boolean | — | Yes | `false` | Whether the software auto-selects the wash technique. It is possible that auto-select may result in choosing a technique not suitable for washing. When possible, set `autoSelectPrototype` to `false` and explicitly set `prototype` to a wash technique. |
| `prototype` | string | — | Yes (key must be present; may be `""`) | `""` | Explicit technique name. The **key must be present**, even when `autoSelectPrototype` is `true`. Provide a real technique name unless auto-selecting or supplying `customPrototype`, in which case an empty string is acceptable but the key must still appear. When selecting a technique, choose one intended for washing. Typically wash techniques have "Wash" in their names. |
| `customPrototype` | object (`_biomekType: "technique"`) | — | No | Not present | Inline technique object; when present overrides `prototype`/`autoSelectPrototype`. |
| `operation` | string | — | Yes (persisted by UI) | `"Mix"` | Always `"Mix"`. |

> **Keys deliberately NOT present.** The shared pipetting editor keys (`aspirateTime`, `customHeight`, `pattern`, `washSettleTime`, `selectionInfo`, `useExpression`, `sectionExpression`, `refreshTips`, `tipLabwareClass`, `overrideHeight`, `height`, `heightFrom`, `labwareClass`) do **not** appear on this step; do not author them here.

---

## Enumerated / Constrained Values

| Parameter | Allowed Values | Notes |
|-----------|---------------|-------|
| `operation` | `"Mix"` | Fixed. The wash is implemented as a mix. |
| `volume` | positive number string, `"<n>%"`, or `"=<expr>"` | `%` form = percentage of tips' max-volume-used, clamped to tip capacity. `=` form is evaluated as a method expression. |
| `liquidtype` | a solvent liquid-type name | Must match a wash station's stocked solvent. |

---

## Cross-Field Validation Rules

1. `pod` **must** resolve to a Multichannel pod. A Span-8 or Fixed-8 pod will
   produce an error.
2. Tips **must** be loaded, or a no-tips error is raised.
3. Tips **must** be empty (no liquid content), or a tips-not-empty error is raised.
4. **Auto-locate wash station** (empty `location`): if no position has a `"Wash Station"` characteristic, a no-wash-stations error is raised; if wash stations exist but `liquidtype` is empty, a liquid-type-not-specified error is raised; if none stock the requested solvent, a no-station-with-solvent error is raised.
5. **Explicit `location`**: if the position is not a wash station, a location-is-not-wash-station error is raised; if it is a station but `liquidtype` is empty, a liquid-type-not-specified error is raised; if the station's solvent doesn't match, a wash-station-selection-invalid error is raised.
6. `volume` **must** be a non-empty string, or a volume-not-specified error is raised. A computed volume `≤ 0` returns success without washing (no-op).
7. If `autoSelectPrototype` is `false`, then `prototype` **must** be a non-empty string naming a valid technique. If the technique cannot be found, the step raises a technique-not-found error.
8. If `autoSelectPrototype` is `true`, at least one matching technique must exist for the given context (pod type, head type, tip class, labware, liquid type, volume), or the step raises a technique-selection error.

---

## Structural Context

This step is a leaf and does not support subSteps.

---

## Canonical Examples

### Example 1: Editor-typical explicit-station wash (percent volume, named technique)

Uses the editor's default shape: all string-valued volume/mixCount, `autoSelectPrototype: false`, an explicit `prototype`.

```json
{
  "stepType": "Multichannel Wash Tips",
  "parameters": {
    "pod": "Pod1",
    "location": "WS1",
    "liquidtype": "Water",
    "volume": "110%",
    "mixCount": "2",
    "autoSelectPrototype": false,
    "prototype": "MC Active Wash",
    "operation": "Mix"
  }
}
```

### Example 2: Auto-located station, auto-select technique

```json
{
  "stepType": "Multichannel Wash Tips",
  "parameters": {
    "pod": "Pod1",
    "location": "",
    "liquidtype": "Water",
    "volume": "200",
    "mixCount": "3",
    "autoSelectPrototype": true,
    "prototype": "",
    "operation": "Mix"
  }
}
```

### Example 3: Expression-driven volume, cycles, and location

Pattern where the volume, cycle count, and wash position are method expressions resolved at run time.

```json
{
  "stepType": "Multichannel Wash Tips",
  "parameters": {
    "pod": "Pod1",
    "location": "=xTips_WashPosition",
    "liquidtype": "Water",
    "volume": "=TipWashVol",
    "mixCount": "=TipWashCycles",
    "autoSelectPrototype": false,
    "prototype": "MC Active Wash",
    "operation": "Mix"
  }
}
```

---

## Common Mistakes

- Washing non-empty tips: the step requires empty tips. Dispense before washing.
- **Expecting pipetting keys here**: `selectionInfo`, `refreshTips`, `height`/`heightFrom`, `labwareClass`, `pattern`, `aspirateTime`, `washSettleTime` are **not** Wash keys. Authoring them here has no effect.
- **A `"<n>%"` volume that rounds to `≤ 0`**: silently no-ops (returns success without washing) when tips have used very little liquid. An empty-string `volume`, by contrast, errors.
- Explicit `location` with mismatched solvent: raises a wash-station-solvent-mismatch error — the station must both be a wash station *and* stock the requested `liquidtype` (case-insensitive).

---

## Related Guides

- **[Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md)** — technique auto-selection (applies to the internal Mix operation).
- **[Method JSON Structure](../Method-JSON-Structure.md)** — envelope, `_biomekType`/`comArray` encoding, structural rules.
