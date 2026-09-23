# Base Keys & Serialization Conventions

Applies to **every** step. The per-step docs assume this file and do **not**
re-document the universal base keys, the casing rules, or the shared value-encoding
references. Read this once; each step doc covers only its step-specific keys.

> The JSON envelope, step-node structure, `_biomekType` / `comArray`, and tree validation are in
> [Method JSON Structure](../Method-JSON-Structure.md); the shared pipetting/technique keys
> are in [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md).
> This file covers the base-step and serializer keys and the key-name casing rules.

---

## 0. What "Required" means in the parameters table

Per-step docs mark each key `Yes` / `Conditional` / `No` in the Required column. That column
describes what the step needs to **run** (Run or Simulate), not what import checks. Import is
lenient: a method can import cleanly and still be missing a required key — the error surfaces only
when you run or simulate it.

- **Required = Yes** — you must author this key. If it is absent, Run/Simulate fails with a missing-required-key error.
- **Required = Conditional** — needed only in some configurations, because the step reads it only
  when another key has a particular value (e.g. Move Labware's `depth` vs `leaveBottomLabware`, or
  an offset-gripper-only field).
- **Required = No** — safe to omit. When absent, the engine applies the default shown in that key's
  Default column.

### What this means for authoring

Author every **Required = Yes** key explicitly, plus any **Conditional** key your configuration
triggers. You never need to add a **Required = No** key just to be safe — omit it and the engine
supplies the default. When you author a method from scratch, the software uses only the keys you
provide; it does not re-add the editor's other defaults for you, so a missing Required = Yes key is
never filled in automatically.

---

## 1. Key-name casing & spelling (global rule)

Biomek step dictionaries are **case-insensitive** at runtime. Casing only matters for *matching what real exports
look like*, which is what authored methods should emit.

Biomek applies a camelCase rule to each parameter key on export. The conversion lowercases only the **first PascalCase word**: a single
leading uppercase letter is lowercased; a multi-letter leading uppercase run has all but
its last letter lowercased. Everything after the first word boundary — including internal
spaces, underscores, digits, and trailing `?` — is **preserved verbatim**:

| Source constant | Serialized JSON key | Note |
|---|---|---|
| `LiquidType` | `liquidType` | first word `L` → `l` |
| `HelpContextID` | `helpContextID` | first word `H` → `h`; second word `Context` and third word `ID` unchanged |
| `HTML_Dialog` | `htmL_Dialog` | first word `HTML`: all but last lowered → `htmL`; `_Dialog` unchanged |
| `StepUI` | `stepUI` | first word `S` → `s`; second word `UI` unchanged |
| `Use Shake` | `use Shake` | **internal space preserved**; `U` → `u`; `Shake` unchanged |
| `Dynamic?` | `dynamic?` | trailing `?` preserved |
| `WS1` | `wS1` | first word `WS`: all but last lowered → `wS`; digit `1` unchanged |

**Author in this serialized form** (first-letter-lowercase, spaces/`?` intact). Because
matching is case-insensitive, a PascalCase key still *imports*, but it will not
string-match other authored methods or the docs — prefer the serialized spelling everywhere.

> **Note:** matching is fully case-insensitive, so `liquidtype`, `liquidType`, and `LiquidType` all resolve identically. If step examples or exports use an all-lowercase form, it works the same as the camelCase form.

> Exception: keys **inside a SILAS message payload** are case-sensitive (see the SILAS
> step doc and the `_biomekType: "SILASMessage"` note in the JSON-structure reference).

---

## 2. Universal base keys (present on every step; not repeated in the per-step docs)

These keys can appear on essentially any step. They are step **metadata / serializer
bookkeeping**, populated by the framework from the step's registration or editor state.
A method author normally **never sets them**; omit them and the step still imports (the
software supplies its own). Preserve them verbatim when round-tripping an existing
method.

| JSON key | Type | Author sets? | Meaning |
|---|---|---|---|
| `caption` | string | Only to override the label | User caption override shown in the editor |
| `defaultCaption` | string | No (auto) | Built-in/auto-generated caption |
| `helpContextID` | integer | No | Help topic id |
| `editorOptionsMask` | integer | No | Editor options bitmask (drives structure rules) |
| `bitmap` | string | **Yes for Start / Finish**, otherwise No | Step icon reference (e.g., `"OStepUI.ocx,START"`). For **Start** and **Finish**, JSON import replaces the step's parameters wholesale and discards any default value, so the icon breaks unless the key is authored — see the per-step docs. For every other step, omit and the framework supplies its own. |
| `tooltip` | string | No | Tooltip override |
| `orderWeight` | integer | No | Palette sort order (design-time) |
| `palette` | string | No | Editor palette group (design-time) |
| `stepUI` | string (CLSID) | No | Editor UI CLSID override |
| `isPreconfigured` | boolean | No | Step came from a preconfigured template |
| `dynamic?` | boolean | Usually no — see **Exceptions** | Error-handling flag. `false` (the default): the step handles its own enqueue errors. `true`: enqueue errors propagate to the framework instead. Framework-set for most steps — do not author. **Exceptions:** (1) Run Program and Run Procedure carry `dynamic?` explicitly and it must be authored to match the step's own rule (see [Run Program](70-Run-Program.md) and [Run Procedure](66-Run-Procedure.md)); (2) the Fixed-8 pipetting family (Aspirate/Dispense/Mix) writes `dynamic?: true` at construction, so round-trip exports of those steps carry `dynamic?: true`; (3) SILAS reuses the same key name for a step-functional flag with different semantics — blocks until device action completes and enables RDD/DataSet features (see [SILAS](71-SILAS.md)). |

---

## 3. Disabling a step (`disabled`)

Whether the step is disabled (commented out) in the editor. Set the **step-node** boolean `disabled: true` — see the [JSON-structure reference](../Method-JSON-Structure.md) for where it sits on the node. Defaults to `false` and is omitted from export when `false`. When `true`, the step is skipped at run time but kept in the method tree. A `disabled` value other than `true`/`false` fails deserialization. The node-level flag is the entire authoring interface: you never write anything inside `parameters` to disable a step.

**Anchor steps cannot be disabled.** Start, Finish, and any container terminator (`End`,
`Multichannel Select Tips End`, or an add-on container's own terminator) are structural
bookends; `disabled: true` on those is rejected at import.

---

## 4. Keys that look shared but are genuinely per-step

`let`, `weak`, and `prompt` appear on many steps but are **not** base keys — they are real scope
keys with step-specific meaning, documented in the relevant per-step docs. What matters for authoring:

- **Author them on the scope steps that use them** — Start, Let, Run Method, Define Procedure, and
  Run Procedure each carry all three (empty `{}` is fine); author them the same way.
- **Do NOT author them on a `Worklist` step** — its per-row scope is built internally, so
  `let`/`weak`/`prompt` do not belong on a Worklist node.
- `let` alone also drives an advanced runtime-override mechanism on Script and similar steps — see
  [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md#advanced-the-let-override-mechanism) §"The `let` Override Mechanism".

---

## 5. Inert / trap keys policy

Many step dictionaries carry keys that are **UI panel state** (`collapsed`,
`showTransferDetails`, `splitterPosition`, `PanelState*`, …) or **vestigial/dead**
(`podsToMaxZ`, `wizard`, …) — persisted but never read at runtime, or read
only for editor cosmetics. Per-step docs classify each such key explicitly as
**inert (UI-state)** or **vestigial (no runtime effect)** so an author neither relies on
them nor treats their presence in exports as meaningful. Do **not** invent behavior for
a key Biomek does not read.

Steps may carry other keys that are not defined in the step's documented
parameters. **If you encounter a key that isn't documented here**, treat it as
unsupported: preserve it when round-tripping an existing method, but do not
author it or rely on it.
