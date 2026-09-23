# Concept Guides

Cross-cutting Biomek Software concepts the per-step docs assume you are already familiar with. Each guide is self-contained and usable without Biomek source-code knowledge. The step docs stand alone — these guides can be removed without breaking them.

| # | Guide | Description |
|---|-------|-------------|
| 01 | [Expressions](01-expressions.md) | Expression syntax, how a field decides literal vs. expression, and which fields accept expressions. |
| 02 | [Labware & Position Resolution](02-labware-and-position-resolution.md) | Deck position vs. labware class, and the per-step-family key spellings for each. |
| 03 | [Probe & Mandrel Selection](03-probe-and-mandrel-selection.md) | Which pod probes/mandrels are active — Span-8 `spacing` / `useProbes` / `mandrelExpression`, Fixed-8 `numberOfTips`, and the Multichannel Select Tips container rules. |
| 04 | [The Variable Environment](04-variable-environment.md) | How `let`, `weak`, and `prompt` bring named variables into scope for expressions. |
| 05 | [Well Volume Tracking](05-well-volume-tracking.md) | The enqueue-time per-well volume ledger — what seeds it, what debits it, and how to read the "well only has 0.000 uL" and "volume out of allowed range" errors. |

Devices — how an installed peripheral is bound to a deck position, and which steps address one by
position vs. by device name — are covered where the binding is read, in
[Deck Layouts Format](../biomek-file-formats/deck-layouts/deck-layouts-format-spec.md#the-device-block) §"The
`device` block".

The instrument/pod model, techniques (auto-select, family prefixes, height override), liquid & volume types, base-key conventions, and 1-based row-major well numbering are covered in the top-level Foundations documents:

- Instrument variants, pod types, pod-naming rule, and which pod family each step accepts — [Introduction to Biomek and Liquid Handling Method Writing](../Introduction-to-Biomek-Method-JSON.md).
- Techniques, template-vs-technique split, auto-select algorithm, `F8`/`S8`/`MC` family prefixes, Fixed-8 no-auto-select rule, liquid types, volume types, and height override — [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md).
- Base keys, key-name casing, and the `disabled` property — [Base Keys & Serialization Conventions](../step-documentation/00-Base-Keys-and-Conventions.md).
- Well numbering math (`firstWell` / `+WellsX` stride, `=Col` sweep) — [Introduction to Pipetting Steps](../Introduction-to-Pipetting-Steps.md#well-numbering) §"Well Numbering".

Read alongside the format specification (`../Method-JSON-Structure.md`) and the
per-step reference (`../step-documentation/`).
