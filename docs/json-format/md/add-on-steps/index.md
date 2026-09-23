# Add-on Steps

Biomek Software includes a core set of built-in step types, documented in the
[step documentation index](../step-documentation/index.md). Beyond that core set, **add-on
modules, device integrations, and site plug-ins register their own step types**. This area
holds the reference documentation for those steps.

An add-on step behaves like any other step in the method tree — same envelope, same base
keys, same structural rules — with two differences worth knowing before you author one:

- **Its `stepType` may be a friendly name or a COM ProgID, depending on the add-on.** Many
  add-ons publish a short, friendly `stepType`; one that does not carries its registration
  identifier (a ProgID) instead. Import resolves a `stepType` against the built-in
  registry first, then as a CLSID string, then as a ProgID — so a ProgID is a legal `stepType`.
  Check the add-on's own documentation for the exact `stepType` it expects. See *Beyond the
  Built-in Steps* in [Method JSON Structure](../Method-JSON-Structure.md).
- **The add-on must be installed on the target instrument.** An add-on step imports only on an
  install where its library is registered. On an instrument without it, import fails with an
  unresolved-CLSID error. Confirm the target install before authoring one.

Shared conventions (universal base keys, expression syntax, labware/position resolution,
probe/mandrel selection, the variable environment) are the same as for built-in steps and live
in [Base Keys & Serialization Conventions](../step-documentation/00-Base-Keys-and-Conventions.md)
and in [`../concept-guides/`](../concept-guides/index.md).

---

## Documented add-on steps

| Step | `stepType` | Requires | What it does |
|------|-----------|----------|--------------|
| [Guided Setup](87-Guided-Setup.md) | `"Guided Labware Setup"` | Guided Labware Setup add-on | Configures the deck at method start with a guided labware placement dialog; loads a named layout, verifies pod configuration, and pauses for operator confirmation. |

## Add-on steps not documented here

If you need to author an add-on step that this area does not cover, obtain its parameter
reference from Beckman Coulter — or, for a third-party module, from the add-on's vendor —
rather than inferring it from an exported method. An add-on step can carry structural or
parameter requirements that are not evident from a sample export.
