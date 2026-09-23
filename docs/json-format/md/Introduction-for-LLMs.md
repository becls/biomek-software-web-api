# Introduction for LLMs
Reference documentation for authoring Biomek methods in the JSON format.

For general information about Biomek systems, refer to the Biomek i3
Instructions for Use and Biomek i5 and i7 Instructions for Use.

**Read order:**

1. [Introduction to Biomek Method JSON](Introduction-to-Biomek-Method-JSON.md) — what a Biomek is, what a method is, pod/step vocabulary.
2. [Method JSON Structure](Method-JSON-Structure.md) — the JSON envelope, `_biomekType` discriminators, the step catalog, structural validation, and how to handle add-on / plugin steps.
3. [Introduction to Pipetting Steps](Introduction-to-Pipetting-Steps.md) - low-level versus high-level pipetting steps, common keys, and tip handling.
4. [Pipetting Techniques and Templates](Pipetting-Techniques-and-Templates.md) — techniques, template-vs-technique split, technique-family prefixes, auto-select, liquid types, volume types, and height override. Read when authoring any pipetting-family step.
5. [Concept guides](concept-guides/index.md) — cross-cutting models (expressions, labware/position resolution, probe/mandrel selection, variable environment, well-volume tracking) that the per-step docs assume you are familiar with.
6. Per-step reference under [step-documentation/](step-documentation/) — browse the [grouped step index](step-documentation/index.md).

The envelope's `formatVersion` is the **method** JSON format version — the JSON format, not the software release — and is versioned independently of the Instrument Settings export, which carries its own `formatVersion`. See [Method JSON Structure](Method-JSON-Structure.md) for the current value. Author exactly the current version: import accepts only that version and rejects anything else — an older value as *too old*, and any higher or longer value (for example a three-part `"x.y.z"`, which reads as higher) as *too new*.

## Quick reference

| Resource | What it's for |
|----------|---------------|
| [Cheatsheet](cheatsheet.md) | One-page quick reference — envelope, step-node shape, anchor/terminator rules, the well-selection idioms, and the gotchas that bite most often. |
| [Worked Example: Simple i3 Fixed-8 Method](Worked-Example-End-to-End.md) | The smallest complete method: a single-column plate-to-plate transfer on i3/Fixed-8. Start here if you want a minimal working template to modify. |
| [Worked Example: Full i5 Span-8 Method](Worked-Example-Span-8-Multi-Step.md) | A full-scale method on i5/Span-8 — loops over columns, reservoir draws, tip washing, and timed pauses. Read this once the minimal template is no longer enough. |
| [Step documentation index](step-documentation/index.md) | Grouped catalog of every step — by pod family, by category, and by number. |

## Foundations

| File | Description |
|------|-------------|
| [Introduction to Biomek Method JSON](Introduction-to-Biomek-Method-JSON.md) | Instrument family, pod types, method tree structure, and the vocabulary the rest of the docs use |
| [Method JSON Structure](Method-JSON-Structure.md) | How to construct a valid Biomek method in the JSON format Biomek reads and writes |
| [Pipetting Techniques and Templates](Pipetting-Techniques-and-Templates.md) | Biomek pipetting technique system, template selection, height overrides, and `C__` context variables |

## Concept guides

Cross-cutting Biomek Software concepts that the per-step reference assumes you are already familiar with. The step docs are self-contained — these guides can be removed without breaking them; they simply explain the underlying concepts a step doc expects you to know.

| # | Guide |
|---|-------|
| — | [Concept guides index](concept-guides/index.md) — table of contents |
| 01 | [Expressions](concept-guides/01-expressions.md) |
| 02 | [Labware & Position Resolution](concept-guides/02-labware-and-position-resolution.md) |
| 03 | [Probe & Mandrel Selection](concept-guides/03-probe-and-mandrel-selection.md) |
| 04 | [The Variable Environment](concept-guides/04-variable-environment.md) |
| 05 | [Well Volume Tracking](concept-guides/05-well-volume-tracking.md) |

Instrument/pod model, technique auto-selection, liquid & volume types, and base-key conventions are covered in the Foundations documents above ([Introduction to Biomek Method JSON](Introduction-to-Biomek-Method-JSON.md), [Pipetting Techniques and Templates](Pipetting-Techniques-and-Templates.md), and [Base keys and conventions](step-documentation/00-Base-Keys-and-Conventions.md)). Well-numbering / `firstWell` math lives in [Introduction to Pipetting Steps](Introduction-to-Pipetting-Steps.md#well-numbering) §"Well Numbering".

## Instrument & project-item reference

The per-step docs assume you are familiar with your own instrument's configuration. Before authoring, export your instrument's settings from the Biomek software (**File ▸ Export ▸ Instrument ▸ Save As JSON**) and examine them using this guide and the sample/reference material here to interpret what you see. A method that targets a configuration your instrument does not actually have will likely fail to enqueue or run.

That material is a companion volume with its own index — see [Instrument & Project-Item Reference](biomek-file-formats/index.md) for the full listing, including the seven default-instrument exports and the resolved deck views that accompany them. The entry points:

| Resource | What it's for |
|----------|---------------|
| [Pod Settings Format Specification](biomek-file-formats/instrument-settings/pod-settings-format-spec.md) | The pod-settings portion of an Instrument Settings export — envelope, per-pod-class key sets, `$id`/`$ref` refs. |
| [Deck Layouts Format Specification](biomek-file-formats/deck-layouts/deck-layouts-format-spec.md) | How to read a deck out of `deckLayouts.<deckKey>` — per-position keys, the `device` block, and the reference model. |
| [Reachability and Access](biomek-file-formats/deck-layouts/reachability-and-access.md) | How deck position reachability and access constrain which positions a pod can address. |
| [Pod Reach Envelopes](biomek-file-formats/deck-layouts/pod-reach-envelopes.md) | Per-pod X travel envelopes, with an i7 reach table. |
| [ALP Catalog](biomek-file-formats/deck-layouts/catalog-alps.md) | The common default ALP (Automated Labware Positioner) types that can appear in a deck. |
| [Project Items](biomek-file-formats/project-items/project-items-format-spec.md) | Valid values a method references — labware classes and patterns, liquid types, tip classes, techniques, and templates. |

## step-documentation/

Per-step reference docs, numbered sequentially. Every step covered by this documentation set has its own reference file under [`step-documentation/`](step-documentation/); browse the catalog, grouped by pod family and area, in the [step documentation index](step-documentation/index.md).

## add-on-steps/

Reference docs for steps contributed by add-on modules and device integrations, rather than by the core step registry. See the [add-on steps index](add-on-steps/index.md).
