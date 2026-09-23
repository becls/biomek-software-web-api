# Instrument & Project-Item Reference

Reference material describing what Biomek expects for instrument configuration and project
items, so that while developing a JSON method you can understand what your parameters, deck
positions, techniques, and labware must match — and what the software validates against. The
per-step docs assume you are familiar with your own instrument's configuration: export your
instrument's settings from the Biomek software (**File ▸ Export ▸ Instrument ▸ Save As
JSON**) and read them alongside the specs, sample exports, and catalogs here to interpret
what you see. A method that targets a configuration your instrument does not actually have
will likely fail to enqueue or run.

Each subarea is self-contained; read the spec for the format you need, and consult the
neighboring sample or catalog file for a worked reference.

This is one of three companion volumes. The [Biomek JSON Authoring Guide](../index.md)
covers method structure, pipetting techniques, worked examples, and the per-step parameter
reference; [Biomek JSON Add-on Steps](../add-on-steps/index.md) covers step types that are
only available when their add-on is installed.

## instrument-settings/

The JSON export produced by the Biomek Instrument Settings feature — pod settings and deck
layouts, per instrument model. See
[Pod Settings Format Specification](instrument-settings/pod-settings-format-spec.md)
for the pod-settings portion of the format, including a roster of seven representative instrument
configurations (i3, i5 multichannel, i5 span-8, i7 multichannel, i7 span-8, i7 hybrid, i7 dual
multichannel).

## deck-layouts/

The deck-layout portion of an Instrument Settings export — how to read a deck out of
`deckLayouts.<deckKey>` in a Biomek Instrument Settings JSON export. See
[Deck Layouts Format Specification](deck-layouts/deck-layouts-format-spec.md) for the
format spec (including the reference model, `$id`/`$ref` handling, and the per-position record
schema), [ALP Catalog](deck-layouts/catalog-alps.md) for the common default ALP types that can
appear in a deck,
[Reachability and Access](deck-layouts/reachability-and-access.md) and
[Pod Reach Envelopes (i7 worked example)](deck-layouts/pod-reach-envelopes.md) for which pod can
reach which position (per-pod X travel envelopes and an i7 reach table), and
[deck-layouts/samples/](deck-layouts/samples/) for an illustrative resolved view of each
default deck (seven files, one per default instrument).

## project-items/

A planned/future summary format for Biomek project items (labware classes, labware patterns,
liquid types, tip classes, pipetting techniques and templates). Biomek currently emits no JSON
export for project items, so this format documents a derived summary shape rather than a
current export. See
[Project Items Format Specification](project-items/project-items-format-spec.md) for
the format, and the six per-category catalog files
([Catalog: Labware Classes](project-items/catalog-labware-classes.md),
[Catalog: Labware Patterns](project-items/catalog-labware-patterns.md),
[Catalog: Liquid Types](project-items/catalog-liquid-types.md),
[Catalog: Tip Classes](project-items/catalog-tip-classes.md),
[Catalog: Techniques](project-items/catalog-techniques.md),
[Catalog: Pipetting Templates](project-items/catalog-templates.md)) for the defaults included with Biomek.
