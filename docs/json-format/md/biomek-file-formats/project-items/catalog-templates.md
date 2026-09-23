# Catalog: Pipetting Templates

This document describes only the default templates that are included with
Biomek Software. A given Biomek system likely has additional templates
developed by users which are preferred for use over the defaults. Users should
provide the names of available templates on their systems to LLMs for use in
method editing.

> **The names here are real and usable.** Templates live inside the project's template library
> (installed alongside techniques) rather than in any exported file, so this catalog documents the
> **default pipetting templates included with Biomek** as prose — and these are the valid template
> names to reference. A specific project may add its own, so treat this as the default set, not an
> exhaustive list.
>
> `hardwareCompatibility` is **pod-based** (Multichannel / Span-8 / Fixed-8); chassis size
> does not affect it, and a hybrid instrument supports the union of its pods. Every default
> template is pod-family-specific by name convention (`F8 …` / `MC …` / `S8 …`) and
> by the templates its bound techniques carry, so no default template reaches all
> three pods and `["any"]` never appears here.

## Scope and derivation

- **Category:** Pipetting Templates. There is no project-item export today, so templates
  are not written to any file — they live only inside the project (installed alongside
  techniques) and are edited in the Biomek software via **Utilities > Pipetting Template
  Editor**. Method JSON references a template only indirectly, through the `template`
  field of the technique that binds it.
- **Sources:** the four default projects (`Biomek_i3Project`, `Biomek_i5Project-MC`,
  `Biomek_i5Project-Span`, `Biomek_i7Project`) and the pipetting-template definitions each
  one installs into its Pipetting Template Editor.
- **Pod mapping used to derive `hardwareCompatibility`:** each pod tag comes from the
  single-pod project that owns that pod family — Fixed-8 from `Biomek_i3Project`,
  Multichannel from `Biomek_i5Project-MC`, Span-8 from `Biomek_i5Project-Span`. The
  `Biomek_i7Project` (Hybrid) project includes duplicates of the Multichannel and Span-8
  templates but is not used to add tags.
- All default template names are pod-family-prefixed and the corresponding
  techniques bind them within that pod family, so every entry here resolves to a single pod
  (`["Fixed-8"]`, `["Multichannel"]`, or `["Span-8"]`).
- **Technique compatibility:** each template lists the default techniques whose
  `template` field points at it. See `catalog-techniques.md` for the technique payloads
  (each entry includes a `template` field that names the template it binds).

## Method-JSON reach

**Method JSON references techniques, not templates.** No pipetting step, and no
`_biomekType` value, refers to a template — you cannot author, edit, or select a template
from method JSON, and you cannot change the template a technique binds from method JSON
either (the technique's `template` field is a read-only property of the technique). The
runtime path is:

1. The pipetting step names a technique (via `prototype`, or auto-select, or an inline
   `customPrototype` payload).
2. The technique's `template` field names the pipetting template it binds.
3. The template's ordered sequence of pipetting actions executes, reading its parameter values
   from `C__` context variables the step and technique populate at run time.

To change the physical procedure, edit the template in the Pipetting Template Editor, or
pick a technique whose template already does what you need. See
`Pipetting-Techniques-and-Templates.md` for the full technique/template model, the C__
variable list, and the auto-select algorithm.

## Items

### Fixed-8 templates (`hardwareCompatibility: ["Fixed-8"]`)

All Fixed-8 pipetting templates are included in `Biomek_i3Project` only.

#### `F8 Pipetting`

- **What it does:** the general-purpose Fixed-8 pipetting recipe — the aspirate/dispense/mix
  operation tree used by the great majority of Fixed-8 techniques. Each operation begins
  with an axes move to the pipetting position, then executes an air-gap, a mix, an
  aspirate/dispense, a trailing air-gap, and an optional tip-touch — all parameterized on
  `C__` variables so the bound technique's General/Aspirate/Dispense/Mix tab values drive
  the actual motion.
- **Technique compatibility:** bound by `F8 High`, `F8 Low`, `F8 Medium`,
  `F8 Medium TopDispense`, and `F8 MultiDispense`.
- **Where to see the full recipe:** open `F8 Pipetting` in **Utilities > Pipetting Template
  Editor** in the Biomek software; that view is authoritative for the action sequence,
  variable bindings, and any conditional guards.

#### `F8 Pipetting TipDipDispense`

- **What it does:** a Fixed-8 pipetting variant that adds a post-blowout "tip dip" — a
  short dispense-only move to 0 mm from the liquid, guarded on the labware having a
  known/nominal liquid volume — to knock residual droplets off the tip on the way out.
  Otherwise mirrors `F8 Pipetting`.
- **Technique compatibility:** bound by `F8 Medium TipDipDispense`.
- **Where to see the full recipe:** open `F8 Pipetting TipDipDispense` in **Utilities >
  Pipetting Template Editor** in the Biomek software.

### Multichannel templates (`hardwareCompatibility: ["Multichannel"]`)

All Multichannel pipetting templates are included in `Biomek_i5Project-MC` and `Biomek_i7Project`.

#### `MC Pipetting`

- **What it does:** the general-purpose Multichannel pipetting recipe — the aspirate,
  dispense, and mix operation tree that carries the great majority of 96- and 384-format
  multichannel work. All motion parameters read from `C__` variables so the bound
  technique's tab values drive the actual behavior.
- **Technique compatibility:** bound by `MC`, `MC MultiDispense`, `MC P60`,
  `MC P300 High`, and `MC P1200 High`.
- **Where to see the full recipe:** open `MC Pipetting` in **Utilities > Pipetting Template
  Editor** in the Biomek software.

#### `MC Low-Volume Pipetting`

- **What it does:** the Multichannel low-volume pipetting variant, tuned for reliable
  aspirate/dispense at very small volumes (single-digit uL). Uses a hard-coded blowout
  speed rather than reading `C__AspirateSpeed` for the post-aspirate air gap, to keep
  ejection dynamics predictable near the head's low-volume floor. Otherwise mirrors the
  `MC Pipetting` structure.
- **Technique compatibility:** bound by `MC Low` and `MC P60 Low`.
- **Where to see the full recipe:** open `MC Low-Volume Pipetting` in **Utilities >
  Pipetting Template Editor** in the Biomek software.

#### `MC Active Washing`

- **What it does:** the Multichannel wash-station recipe. Aspirate is blocked — aspirating
  from a wash station is not supported — and dispense targets the wash-station waste area
  rather than a well, so mix and tip-touch settings on the technique's Dispense tab are
  disregarded by design.
- **Technique compatibility:** bound by `MC Active Wash`.
- **Where to see the full recipe:** open `MC Active Washing` in **Utilities > Pipetting
  Template Editor** in the Biomek software.

### Span-8 templates (`hardwareCompatibility: ["Span-8"]`)

All Span-8 pipetting templates are included in `Biomek_i5Project-Span` and `Biomek_i7Project`.

#### `S8 Pipetting`

- **What it does:** the general-purpose Span-8 pipetting recipe — the aspirate, dispense,
  and mix operation tree that drives most Span-8 work with disposable or fixed tips. All
  motion parameters read from `C__` variables so the bound technique's tab values drive the
  actual behavior.
- **Technique compatibility:** bound by `S8 250`, `S8 250 Low`, `S8 1000 Low`,
  `S8 1000 Medium`, `S8 1000 High`, and `S8 MultiDispense`.
- **Where to see the full recipe:** open `S8 Pipetting` in **Utilities > Pipetting Template
  Editor** in the Biomek software.

#### `S8 Active Washing`

- **What it does:** the Span-8 wash-station recipe. Aspirate is blocked — aspirating from a
  wash station is not supported — and dispense targets the wash-station waste area rather
  than a well, so mix and tip-touch settings on the technique's Dispense tab are
  disregarded by design.
- **Technique compatibility:** bound by `S8 Active Wash`.
- **Where to see the full recipe:** open `S8 Active Washing` in **Utilities > Pipetting
  Template Editor** in the Biomek software.

#### `S8 Clot Detection`

- **What it does:** the Span-8 clot-detection recipe — an `S8 Pipetting`-style tree
  augmented to run the clot-detection routine during aspirate, using the technique's
  Clot Detection tab settings (`C__CDConductivity`, `C__CDHeight`, `C__CDRepeat`,
  `C__CDSpeed`, `C__CDErrorAutoRecover`, and related keys). Only meaningful with an
  LLS-capable tip, and typically paired with the tip list that `S8 Clot Detection` scopes.
- **Technique compatibility:** bound by `S8 Clot Detection`.
- **Where to see the full recipe:** open `S8 Clot Detection` in **Utilities > Pipetting
  Template Editor** in the Biomek software.

#### `S8 Septa Piercing`

- **What it does:** the Span-8 septum-piercing recipe used with fluted septum-piercing
  tips on septum-capped tube racks. Tip touch is disabled by the template because no
  lateral movement is allowed while a tip is inside a septum tube; the mix heights on the
  Aspirate and Dispense tabs are also disregarded by the template for the same reason; and
  the template does not follow the liquid level, to avoid extra motions inside the tube.
- **Technique compatibility:** bound by `S8 SeptaFluted`.
- **Where to see the full recipe:** open `S8 Septa Piercing` in **Utilities > Pipetting
  Template Editor** in the Biomek software.

## Note on user-defined templates

Users can create their own pipetting templates in **Utilities > Pipetting Template Editor**
alongside the defaults listed above. This catalog covers only the Beckman
defaults; the installed template set on a specific instrument may include additional
user-defined templates that are outside the scope of this reference.
