# Pipetting Techniques, Templates, and Auto-Selection

**Applies to** any pipetting-family step (Span-8 Aspirate/Dispense, Multichannel Aspirate/Dispense/Mix, Transfer, Combine, etc.).

> **Not a Biomek export — these are the defaults.** Pipetting techniques and templates
> are a kind of **project item**, and Biomek has **no JSON export for project items** today.
> Everything described here therefore documents the **defaults included with Biomek**, not an
> exported file. In practice most users work with **their own** techniques and templates (and
> other project items) that they have created or tuned for their assays, rather than the default
> set — so treat the technique names and values shown here as illustrative examples, and
> confirm the actual set installed on your target instrument. For everyday authoring you only need
> to reference an existing technique **by name** (one already defined in your project), or leave
> auto-selection on. Hand-authoring the technique *configuration* in JSON — the height overrides and
> the `C__` pipetting context variables covered below — is an **advanced** capability most authors
> never need; techniques are normally created and tuned in the Biomek software, not written by hand.
> Pipetting **templates** cannot be authored in JSON at all.

---

## Terminology

The JSON key names and the Biomek UI use different terms for the same concepts:

| JSON key | UI / user-facing term | What it is |
|---------------|----------------------|------------|
| `Prototype` (step dictionary key) | **Technique name** | The string name of the selected technique |
| `AutoSelectPrototype` | **Auto-Select technique** | Whether the software picks the best technique automatically |

---

## Templates vs. Techniques

A **pipetting template** is the parameterized *recipe*: an ordered program of atomic physical operations (aspirate, dispense, mix, air-gaps, tip-touch, …) whose values are left open as `=C__` variable placeholders. It is pod-family-specific — a template such as `F8 Pipetting`, `S8 Pipetting`, or `MC Pipetting` runs only on its own pod family — and a single template is not tip- or volume-specific, so one template serves many techniques across that family's regimes. Each family also has several templates for procedures that differ physically (base pipetting plus variants such as septa-piercing, clot-detection, active-wash, and low-volume).

A **technique** is a *named binding* on top of a template: it picks one template and fills in concrete values for that template's variables, plus the LLS / clot / piercing / liquid-type / calibration settings, scoped to a specific pod family + tip / volume / liquid regime (min/max volume, rank).

Think of the template as a **function definition with parameters** and the technique as a **named partial application** of it — which is why many techniques can bind the same template (several `MC …` techniques in the default project drive `MC Pipetting` with different values). A technique without a template, or a template without a technique, does nothing.

**The one rule for JSON authors:** method JSON references **techniques**, never **templates**. There is **no JSON representation for a pipetting template** — you cannot create one, edit its operation tree, or point a step at one in authored JSON. Templates live only inside the project and are reached *indirectly*, as the thing a technique selects. The [JSON Support Boundary](#json-support-boundary) below states this precisely.

### Where to work with these in Biomek

You do not author techniques or templates in JSON — you configure them in the Biomek software. If you need to:

- **see which techniques a project provides** (and the pod family, tips, and volume range each targets), open the **Technique Browser** (`Utilities ▸ Technique Browser`);
- **view or adjust a technique's settings** (and see which template it uses), open it in the **Technique Editor**;
- **inspect or edit a template's operation tree**, open the **Pipetting Template Editor** (`Utilities ▸ Pipetting Template Editor`).

### JSON Support Boundary

Method JSON references **techniques**, never **templates**. A step can name a technique, auto-select one, or embed an inline technique object — the three modes are detailed under [Technique Selection](#technique-selection) below.

Templates sit entirely outside that boundary: no step and no `_biomekType` value refers to a template, so you cannot author one, edit its operation tree, or even change *which* template a technique uses from JSON. To change the physical procedure, edit the technique's template in the Pipetting Template Editor, or pick a different technique whose template already does what you need.

---

## How Techniques Work

### Layers

1. **Pipetting step** (e.g., Multichannel Aspirate) — configured in the method tree. Contains keys like `pod`, `location`, `volume`, `autoSelectPrototype`, `prototype`, `overrideHeight`, etc.
2. **Technique** — stores all the detailed pipetting parameters (aspirate speed, dispense speed, heights, blowout settings, tip touch, prewet, LLS, clot detection, etc.). Selected either automatically or by name.
3. **Pipetting template** — an ordered sequence of physical actions (aspirate, dispense, tip touch, mix, etc.) that the instrument physically executes. Each action reads its parameters from `C__` context variables populated by the technique.

### Execution Flow

1. The step resolves a technique (auto-select or by name).
2. The step injects `C__` context variables into the runtime scope — some from step properties, some from the technique.
3. The step calls the technique's template operation (Aspirate, Dispense, or Mix).
4. The template's actions execute in order, reading `C__` variables for their parameters.

---

## Technique Selection

### Auto-Select Mode (`autoSelectPrototype: true`)

When auto-select is enabled, the step builds a context from the current state and finds the best-matching technique.

**Inputs to auto-selection**:

| Input | Source |
|-------|--------|
| Pod type | the pod's configured type |
| Head type | the pod's head type |
| Tip class | the class of the tips loaded on the pod |
| Labware class | From the step's `labwareClass` property |
| Liquid type | Resolved from the step's `liquidType` — may come from well contents, tip contents, or the literal value |
| Volume | From the step's `volume` — or computed from tip contents when emptying tips |

**If no technique matches**, enqueue fails with a technique-not-found error.

### Named Technique Mode (`autoSelectPrototype: false`)

When auto-select is disabled:

1. If `customPrototype` is present (an inline `_biomekType:"technique"` object), it is used directly and overrides `prototype`. It is a valid — if uncommon — thing to author, and must be preserved when round-tripping an existing method; the editor usually creates it.
2. Otherwise, `prototype` is looked up via the technique lookup (by name). The lookup is attempted **twice**: first against the raw (unevaluated) string, then, if that fails, against the expression-evaluated string. So `prototype` may either be a literal technique name or an expression that resolves to one at runtime.
3. If both lookups fail, enqueue fails with a technique-not-found error.

### Which Mode to Use

| Scenario | Recommendation |
|----------|---------------|
| General-purpose method, user has techniques configured | `autoSelectPrototype: true` — let the system pick the best match |
| Method requires a specific technique | `autoSelectPrototype: false`, set `prototype` to the exact technique name |
| Unknown technique availability | `autoSelectPrototype: true` — adapts to what's installed |

The default for `autoSelectPrototype` is `false`. This means if you don't set it, the step will look for a named technique in `prototype` — which will fail if `prototype` is empty. Prefer to set `autoSelectPrototype` explicitly.

**Fixed-8 exception**: The Fixed-8 pipetting steps (`Fixed-8 Aspirate`, `Fixed-8 Dispense`, `Fixed-8 Mix`) **do not support auto-selection at all** — this is a permanent, intentional design choice. Setting `autoSelectPrototype: true` on a Fixed-8 step fails at enqueue. For Fixed-8, always use `autoSelectPrototype: false` with an explicit `prototype` (or a `customPrototype` object). Auto-select applies only to Span-8 and Multichannel steps.

### Notes On Choosing Techniques

Techniques that are intended to do LLS or clot detection are only compatible
with conductive tips or fixed tips on the Span-8 pod.

Techniques with "wash" in the name are typically intended for tip washing, not
for regular liquid transfers.

Techniques sometimes include the intended volume in the name (e.g. "low" or
"high" or "medium"). For transfers of low volumes, choose a low volume
technique when available. For transfers of high volumes, choose a high volume
technique when available. For all others, choose a medium technique.

### Inline Technique Object (`customPrototype`)

A step may also carry an inline technique object under the `customPrototype` key — a JSON
object with `_biomekType: "technique"` embedded directly in the step's parameters rather than
a named reference resolved from the project. When present, `customPrototype` is used directly
and **overrides** `prototype` at enqueue.

This mode is uncommon and not recommended in hand-authored JSON — the Biomek editor is what usually creates a
`customPrototype`, typically when the user tunes technique settings on a specific step without
saving them back as a named project technique. It is nonetheless a valid, authorable JSON
parameter, and it **must be preserved when round-tripping** an existing method. Do not strip
it on the assumption that it is UI-only state.

**Set `autoSelectPrototype: false` alongside it.** On Span-8 and Multichannel steps the inline
object is only consulted when auto-selection is off; leaving `autoSelectPrototype: true` there means
the step auto-selects a named technique and your inline object is ignored. On Fixed-8 auto-selection
is rejected outright, so `false` is required anyway.

**An inline technique must be complete.** It is not a patch over a named technique: nothing is
merged in and no defaults are filled: the object you author becomes the whole technique. Omitting a
block the step needs fails at enqueue — a missing `general` block, for instance, fails with
`The technique does not implement the General settings.` The reliable way to author one is to copy a
full technique payload for your pod family from
[Catalog: Techniques](biomek-file-formats/project-items/catalog-techniques.md) and change only the
keys you want, leaving the rest intact.

The keys worth knowing when tuning one:

| What you want to change | Key | Where it nests |
|---|---|---|
| Aspirate / dispense speed | `c__Speed` | in the `aspirate` and `dispense` blocks respectively — this is the one to edit on every pod family |
| Aspirate / dispense speed (Fixed-8 only, additional) | `c__aspiratespeed`, `c__dispensespeed` | the `general` block. Fixed-8 techniques carry these as well; Multichannel and Span-8 techniques do not. |
| Mix after dispensing | `c__Mix` (on/off), `c__MixCount`, `c__MixVolume`, `c__MixAspirateHeight` / `c__MixAspirateFrom` / `c__MixAspirateSpeed`, and the matching `c__MixDispense*` trio | the `dispense` block |
| Mixing as the step's own operation | the same `c__Mix*` keys minus `c__MixCount` and `c__MixVolume` | the `mix` block. A Mix step supplies count and volume itself from its `mixCount` and `volume` parameters, overriding whatever the technique carries. |

The editor writes `prototype: "[Custom]"` next to a `customPrototype` it created, matching the
inline object's `name`. That is harmless — the inline object wins — and should be preserved on
round-trip.

### Technique-Family Rule (F8 / S8 / MC prefixes)

Techniques are **pod-family specific**. In practice, technique names follow a
prefix convention that maps to the pod they were configured for:

| Prefix | Pod family | Examples |
|--------|-----------|-------------------------------------------|
| `F8 …` | Fixed-8 | `F8 High`, `F8 Low`, `F8 Medium`, `F8 MultiDispense`, `F8 Medium TopDispense` |
| `S8 …` | Span-8 | `S8 250`, `S8 1000 High`, `S8 Active Wash`, `S8 Clot Detection`, `S8 SeptaFluted` |
| `MC …` | Multichannel | `MC P60`, `MC P300 High`, `MC P1200 High`, `MC MultiDispense`, `MC Active Wash` |

**Naming a technique from the wrong family on a step is a common porting failure** — e.g.
leaving `prototype: "F8 Medium"` on a `Span-8 Aspirate` after converting a Fixed-8 method
to Span-8. The `_autoSelect_` category machinery further scopes each technique to its
`Pod.Type`, so a cross-family reference is either rejected at
auto-select time or, when named explicitly, mis-renders the step in the editor even when
the underlying JSON is otherwise valid.

**When you change `pod` or the pod family of any pipetting step, review every
`prototype` (and any inline `customPrototype`) and re-point them at same-family techniques
as needed**. Prefer confirming the actual technique names available on the target
instrument, rather than guessing.

The prefix is a naming convention — the *authoritative* pod-family binding lives inside
the technique's `Pod` and `_autoSelect_` sub-dictionaries.

> **Technique names are project-specific.** The names in the table above (e.g. `F8 Medium`,
> `MC P60`, `S8 1000 Medium`) are the defaults in Beckman's example projects
> (`Biomek_i3Project`, etc.). A given customer project may define entirely different technique
> names, or may not include some of those defaults. A `prototype` value must name a technique
> that **actually exists in your project** — do not assume a default catalog name is present.
>
> To find the valid technique names for your project, open the **Technique Browser** in the
> Biomek software (`Utilities ▸ Technique Browser`). It lists exactly the techniques your
> project defines. If you do not have that list, obtain it from whoever owns the project
> before authoring method steps.
>
> Because **Fixed-8 never auto-selects**, every Fixed-8 Aspirate / Dispense / Mix step **must** name a `prototype` that exists in the project (or supply an inline `customPrototype`) — otherwise enqueue fails with: `No technique is defined with the name '<name>'`. Multichannel and Span-8 steps can instead set `autoSelectPrototype: true`.
>
> If you cannot rely on a specific named technique being present, use `customPrototype` — an
> inline technique object (`_biomekType: "technique"`) embedded directly in the step — as the
> project-independent alternative. See [Inline Technique Object](#inline-technique-object-customprototype)
> above for details.

---

## Liquid Types

Every aspirate/dispense/mix operation has a `liquidType`. It feeds technique auto-selection (as one of the match criteria) and records what ends up in the tips and wells. There are two special tokens plus any number of named liquids:

| `liquidType` value | Meaning | Valid on |
|--------------------|---------|----------|
| `"Well Contents"` | Auto-detect the liquid from the labware's tracked contents at run time. | **Aspirate / Mix (source) only** |
| `"Tip Contents"` | Use whatever the tips are currently known to hold. | **Dispense (destination) only** |
| A **named liquid** (e.g. `"Water"`, `"Serum"`) | Use this specific liquid type explicitly, overriding auto-detection. | Any operation |

Rules to watch:

- `"Tip Contents"` on an **aspirate or mix** step fails at run time — you cannot aspirate "the contents of the tips." Use `"Well Contents"` or a named liquid.
- `"Well Contents"` on a **dispense** step fails at run time (symmetric to the rule above) — a dispense is defined by what the tips hold, not the destination well's contents. Use `"Tip Contents"` or a named liquid.
- A **named** liquid must be one the project knows about; it participates in auto-select as one of the match criteria (see §"Auto-Select Mode" above).

In a Transfer/Combine step the `liquidType` lives on each *item*: source items typically carry `"Well Contents"`, destination items `"Tip Contents"`.

> **Liquid Types are project-specific.** The default liquid types provided by Biomek Software are `"Serum"` and `"Water"`
> only. A given customer project may define entirely different liquid types, or
> may not include some of these defaults.
>
> To find the valid liquid types for your project, open the **Liquid Type
> Editor** in the Biomek software.

### Volume types and locatable surfaces

When labware is placed on the deck (in **Instrument Setup**), each piece can declare how well its per-well volumes are known, via `volumeType`:

| `volumeType` | Meaning | Effect at pipetting time |
|--------------|---------|--------------------------|
| `"Unknown"` | Volumes are not known. **(default if omitted)** | A liquid-relative move needs the surface located at run time. On **Span-8**, an LLS technique can sense the surface live even when the volume is unknown. **Fixed-8** and **Multichannel (96- or 384-channel)** pods do not have LLS, so on those pods an `"Unknown"` well fails a liquid-relative move — declare at least `"Nominal"`. |
| `"Nominal"` | Volumes are approximate/estimated. | Surface is *locatable* from the estimate, but LLS (if available) still refines it. |
| `"Known"` | Volumes are definitively known. | The surface is computed from the tracked volume; LLS is skipped. |

> **Pod-family carve-out.** The "LLS rescues an `"Unknown"` well" behavior is **Span-8 only**. On Fixed-8 and Multichannel (96- or 384-channel) pods, a liquid-relative move against an `"Unknown"` well always fails at run time regardless of the technique — LLS does not lift the restriction there. The minimum you can declare is `"Nominal"`.

The single `volumeType` string is authoritative. Matching is **case-insensitive** on import — `"Unknown"`, `"unknown"`, `"NOMINAL"`, `"known"`, etc. all parse to the same enum values. Any other **present** value — an unrecognized string, a numeric string like `"1"`, or a non-string JSON value — throws a `JsonException` on import. **Omitting** `volumeType` entirely is allowed and defaults to `Unknown`.

### Pair `"Known"` / `"Nominal"` with a fill

Declaring a volume type is not the same as *stating the volume*. To actually track contents you also supply, on the labware in Instrument Setup:

- `evalAmounts` — the per-well volumes (µL), and
- `evalLiquids` — the per-well liquid types.

Setting `volumeType: "Known"` **without** an `evalAmounts` fill leaves the tracked well volume at 0 — an aspirate then fails with `Cannot pipette <v> uL; the well only has 0.000 uL in it.` Always pair `"Known"` (or `"Nominal"`) with `evalAmounts` (and `evalLiquids` for the liquid identity). The tracked volume is a running ledger maintained across the whole method, so this error can also mean an earlier step drained the well rather than that the seed is missing — see [Well Volume Tracking](concept-guides/05-well-volume-tracking.md).

---

## Height Override

The step can override the technique's pipetting height on a per-step basis.

### Keys

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `overrideHeight` | boolean | `false` | Whether to override the technique's height for this step |
| `height` | number or expression | `0` | Height offset value (only used when `overrideHeight` is `true`) |
| `heightFrom` | integer or string | `0` | Reference point for the height (only used when `overrideHeight` is `true`) |

### `heightFrom` Values

| Value | String alias | Meaning |
|-------|-------------|---------|
| `0` | `"Liquid"` | Height measured from the liquid surface |
| `1` | `"Bottom"` | Height measured from the bottom of the labware |
| `2` | `"Top"` | Height measured from the top of the labware |

### Sign Conventions

- From **liquid** (`0`): negative values go below the surface (common for aspirating submerged).
- From **bottom** (`1`): positive values go above the bottom (common for avoiding dead volume).
- From **top** (`2`): negative values go below the top rim (common for dispensing into a well).

### Liquid-relative heights need a locatable liquid surface

When a step positions relative to the **liquid surface** — `heightFrom = "Liquid"` (`0`), or a technique
with follow-liquid enabled (`C__FollowLiquid = True`, the default for the stock "follow" techniques) —
the target well's surface must be **locatable at runtime**, via one of:

- a **known/nominal volume**: the labware is placed in Instrument Setup with `volumeType: "Known"` (or
  `"Nominal"`) **and** an `evalAmounts` fill (µL per well) — plus `evalLiquids` for the liquid type; or
- **liquid-level sensing (LLS)** — Span-8 only: LLS-capable tips + an LLS technique, so the level is sensed at runtime. **Fixed-8** and **Multichannel (96- or 384-channel)** pods do **not** have LLS, so this alternative does not apply on those pods — an `"Unknown"` well there still needs `"Nominal"` or `"Known"`.

If neither holds — an `"Unknown"` well with no locatable surface (and, on Fixed-8/96, any `"Unknown"` well, because those pods lack LLS entirely) — the step fails at **runtime** (not enqueue) with a "liquid amounts not defined" error.

`heightFrom = "Bottom"`/`"Top"`
never triggers this — overriding to one of those avoids the error but changes where the tip goes.

The `volumeType` string values are `"Unknown"`, `"Nominal"`, and `"Known"`. See §Volume types above for the per-value semantics and the Fixed-8/96 carve-out.

### How Override Works at Runtime

When `overrideHeight` is `true`:

- `C__Height` is set from the step's `height` property (instead of inheriting from the technique).
- `C__HeightFrom` is set from the step's `heightFrom` property.
- The template's actions use these overridden values for their pipetting height.

When `overrideHeight` is `false`:

- `C__Height` and `C__HeightFrom` are NOT set by the step.
- The template's actions inherit these values from the technique's own height settings.

### Height Encoding Keys

| Key | Description |
|-----|-------------|
| `customHeight` | Encoding discriminator for `height` / `heightFrom`: when `false`, `height` is a numeric offset and `heightFrom` the numeric enum; when `true`, `height` is instead a text string (a typed/custom height, which may be an expression), and `heightFrom` stays the numeric enum for `Liquid` / `Bottom` / `Top` but is written as text for anything else. `true` implies `overrideHeight` is also `true`. For authored JSON with numeric heights, set `customHeight: false`. |

**`customHeight` means the same thing on every step that carries `height` / `heightFrom`.** It is
this one encoding discriminator — not a per-step flag that means something different from step to
step, and not a record of how the value was entered in the editor. What varies by step is only
whether the key has to be present:

- **Required** in each `aspirateProperties` / `dispenseProperties` /
  `diluentAspirateProperties` / `diluentDispenseProperties` sub-dictionary of
  [`Span-8 Serial Dilution`](step-documentation/14-Span-8-Serial-Dilution.md), on the same terms
  as `height`, `heightFrom`, and `overrideHeight` there.
- **Optional everywhere else**, defaulting to `false`. Preserve it on round-trip when present.

No step branches on `customHeight` beyond the encoding it selects, so with numeric heights `false`
is always the right value.

---

## C__ Pipetting Context Variables

These variables are injected into the runtime `Let` dictionary during pipetting step execution. The template's actions read them via expressions like `=C__Volume`.

> **Advanced — authoring these directly is not recommended.** The `C__` variables are an internal
> mechanism that the technique and template system manages for you. This section is reference
> material for understanding runtime behavior; you should not normally set `C__` variables by hand
> in method JSON. Configure pipetting behavior by selecting an appropriate **technique** (or
> adjusting one in the Technique Editor) instead — see the note at the top of this document.

### Step-Injected Variables

These are set by the pipetting step itself from its dictionary properties and the resolved runtime state:

| Variable | Type | Source | Description |
|----------|------|--------|-------------|
| `C__Pod` | string | `pod` property | Pod name |
| `C__Position` | string | Resolved `location` | Deck position string |
| `C__Volume` | number | `volume` property | Pipetting volume in µL |
| `C__RepeatCount` | number | Hard-coded `1.0` | Repeat count (always 1 for single-operation steps) |
| `C__LiquidType` | string | Resolved liquid type | Liquid type name |
| `C__RealWellX` | number | Pod X position | Physical X coordinate of the well |
| `C__RealWellY` | number | Pod Y position | Physical Y coordinate of the well |
| `C__RealWellZ` | number | Calculated Z | Physical Z coordinate for pipetting |
| `C__LabwareWellsArray` | array | Well indices | Array of well indices being targeted |
| `C__LabwareWellDups` | boolean | Hard-coded `false` | Whether wells are duplicated |
| `C__Operation` | string | Operation name | `"Aspirate"`, `"Dispense"`, or `"Mix"` |
| `C__Height` | number | `height` property | Only when `overrideHeight` is `true` |
| `C__HeightFrom` | integer | `heightFrom` property | Only when `overrideHeight` is `true` |
| `C__ConditioningExcess` | number | Read from technique General dict, injected by step | Conditioning excess volume (may be % or µL) |

### Technique-Provided Variables

These come from the technique's stored parameters (not from the step's parameters). They are set by the template system, not by the step:

| Category | Variables |
|----------|-----------|
| **Speed** | `C__Speed`, `C__AspirateSpeed`, `C__DispenseSpeed` |
| **Height** | `C__Height`, `C__HeightFrom`, `C__MinimumHeight` (when not overridden by step) |
| **Blowout** | `C__Blowout`, `C__BlowoutVolume`, `C__BlowoutDelay` |
| **Tip Touch** | `C__TipTouch`, `C__TipTouchDelay`, `C__TipTouchSpeed`, `C__TipTouchHeight`, `C__TipTouchHeightFrom`, `C__TipTouchTheta` |
| **Prewet** | `C__Prewet`, `C__PrewetOverage`, `C__PrewetDelay` |
| **Mix** | `C__Mix`, `C__MixCount`, `C__MixVolume`, `C__MixAspirateHeight`, `C__MixAspirateFrom`, `C__MixAspirateSpeed`, `C__MixDispenseHeight`, `C__MixDispenseFrom`, `C__MixDispenseSpeed` |
| **Follow Liquid** | `C__FollowLiquid` |
| **Trailing Air Gap** | `C__TrailingAirGap`, `C__TrailingAirGapVolume` |
| **Delays** | `C__AspirateDelay`, `C__DispenseDelay` |
| **Calibration** | `C__CalibrationSlope`, `C__CalibrationOffset`, `C__CutoffVelocity` |
| **Conditioning** | `C__ConditioningSectionVolume`, `C__ConditioningSectionCount` |
| **Liquid Level Sensing (LLS)** | `C__DetectionMode`, `C__Conductivity`, `C__ConductivityInherit`, `C__DetectionSpeed`, `C__DoubleDistance`, `C__ReservoirSafeDepth`, `C__LiquidLevel`, `C__LLSInitial`, `C__LLSInitialHeight`, `C__LLSInitialHeightFrom`, `C__LLSRepeat`, `C__LLSAccept`, `C__LLSFail`, `C__LLSAcceptValue`, `C__LLSFailValue`, `C__LLSEveryTime`, `C__LLSErrorAutoRetry`, `C__LLSErrorAutoRecover` |
| **Clot Detection** | `C__ClotDetectionSpeed`, `C__ClotLimit`, `C__CDHeight`, `C__CDHeightFrom`, `C__CDRepeat`, `C__CDEveryTime`, `C__CDErrorAutoRetry`, `C__CDErrorAutoRecover`, `C__CDSpeed`, `C__CDConductivity`, `C__CDConductivityInherit` |
| **Piercing** | `C__PiercingInitialHeight`, `C__PiercingFinalHeight`, `C__PiercingExitHeight`, `C__PiercingFinalSpeed`, `C__PiercingExitSpeed`, `C__PiercedSpeedLimit` |

### Using C__ Variables in Expressions

Any expression-capable string field in a step's parameters can reference `C__` variables with the `=` prefix. For example:

- `"volume": "=C__Volume * 0.5"` — half the context volume
- `"height": "=C__Height + 1.0"` — 1mm above the context height

This is primarily used within template actions, not in the method-level step parameters. Method-level steps set `C__` variables; they don't typically read them.

### Advanced: The `let` Override Mechanism

If a step's dictionary contains a bound `let` key (an object), all key-value pairs from it are copied into the runtime `Let` dictionary, potentially overriding any `C__` variable. This is an advanced automation feature — do not include in normal authored JSON.

---

## What Authored Steps Need to Know

For **technique selection and height override**, a pipetting step's `parameters` use these keys:

1. **`autoSelectPrototype`** — always set explicitly (`true` or `false`).
2. **`prototype`** — required when `autoSelectPrototype` is `false`; must be the exact name of an installed technique.
3. **`overrideHeight`** — set to `true` only when you need a specific pipetting height.
4. **`height`** and **`heightFrom`** — required when `overrideHeight` is `true`.
5. **`customHeight`** — include as `false` for numeric heights. When `true`, `height` is interpreted as text (a typed/custom height, possibly an expression) instead of a numeric offset.

These are **only** the technique-and-height keys. A pipetting step still needs its own core parameters — the pod, the labware/location, the volume, the well selection, the liquid type, and so on — which are documented per step in [step-documentation/](step-documentation/). The blocks below show just the technique/height keys.

The **auto-select** technique/height keys:

```json
{
  "autoSelectPrototype": true,
  "prototype": "",
  "overrideHeight": false,
  "height": 0,
  "heightFrom": 0,
  "customHeight": false
}
```

The **named-technique** technique/height keys (the required form for Fixed-8, which never auto-selects):

```json
{
  "autoSelectPrototype": false,
  "prototype": "F8 Medium",
  "overrideHeight": false,
  "height": 0,
  "heightFrom": 0,
  "customHeight": false
}
```

You do NOT need to:

- Know or set any `C__` variables — these are populated automatically at runtime.
- Know the template structure — this is internal to the technique.
- Configure individual template-action parameters — these come from the technique.

Note on `customPrototype`: an inline technique object; you rarely hand-author one, but it is
valid JSON, overrides `prototype`, and must be preserved on round-trip.
