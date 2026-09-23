# Concept Guide: Expressions

Many string-valued fields in a method can hold either a **literal value** or an
**expression** that is evaluated at run time. This guide defines the expression syntax, how
a field decides which one it got, and which fields accept expressions.

---

## 1. The `=` prefix

An expression is a string whose **first character is `=`**. Everything after the `=` is
evaluated at run time; the result is coerced to whatever the field needs (a number, a
string, a well index, a boolean array, …).

```jsonc
"amount": "10"        // literal: aspirate 10 µL
"amount": "=dose * 2" // expression: evaluate dose*2 at run time
```

A string **without** a leading `=` is taken verbatim as a literal.
**Condition-valued** fields are the exception: the `If` step's `condition`, for example,
**auto-prefixes** the `=`, so a bare string like `"Volume > 50"` is evaluated as an
expression (a leading `=` is optional but recommended for clarity).
This is why almost every
expression-capable field is typed as a **string in the JSON**, even when it carries a
number — the leading `=` is how the engine tells "the literal number 10" apart from "the
formula that happens to start with a digit." Write numeric values destined for these fields
as quoted strings (`"10"`, not `10`).

## 2. What you can write after the `=`

The expression language supports the usual building blocks:

- **Arithmetic** — `+ - * /`, parentheses, e.g. `"=(col - 1) * 2 + 1"`.
- **Variable references** — any name in scope: loop counters, `let`/`weak` bindings,
  globals, prompted values. E.g. `"=col"`, `"=aspirateVolume"`, `"=podID"`.
- **Member access** — dotted names where the runtime exposes them, e.g.
  `"=thisSpanPod.name"`.
- **Array literals** — square-bracket lists, used where a field expects a list (e.g. a
  probe-selection expression: `"=[1,2,3,4,5,6,7,8]"`).
- **Pipetting context variables** (`C__…`) — engine-injected names that appear inside
  pipetting technique/template payloads, e.g. `"=C__Volume * 0.5"`. A pipetting step
  populates them from its own properties and the selected technique, and the actions in the
  technique's template read them at run time; a few also surface in method-level conditions
  when an enclosing pipetting/technique scope injects them (e.g. an `If` condition on
  `=C__…`). You do **not** hand-author them, and you rarely reference them in ordinary
  method-level fields. See [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md).

Where a variable comes from is governed by scope — see [Concept Guide: The Variable Environment (`let` / `weak` / `prompt`)](04-variable-environment.md) for
`let` / `weak` / `prompt` and globals.

## 3. Fields gated by an explicit "use expression" flag

Some fields have a *paired boolean* that selects between two different keys: a literal key
and an expression key. When the flag is **off**, the literal key is read and the expression
key is ignored; when **on**, the reverse. Setting the `=` string alone is not enough — you
must also flip the flag.

For example, Span-8 Aspirate has a flag key `useWellExpression`, which, when set
to `true`, tells the engine to read `firstWellExpression` and ignore `firstWell`
to determine well selection. It also has flag key `useExpression`, which, when
set to `true`, tells the engine to read `mandrelExpression` and ignore
`useProbes` to determine probe/mandrel selection.

```jsonc
// Iterate plate columns via a loop counter Col:
"useWellExpression": true,
"firstWellExpression": "=Col",   // read
"firstWell": 1                   // ignored while the flag is true
```

Refer to individual step documentation for the keys available to the step and
their interactions, as they are not all the same.

## 4. Fields that take an expression directly (no flag)

Most expression-capable fields have **no** gating flag — they simply accept either a
literal string or a `=`-prefixed string in the same key. Common examples across the
pipetting steps:

- Volumes: `amount`, `amounts` (per-probe array of strings), item-level `volume`.
- Counts: `repeats`, `replicates`, `numberOfTips`.
- Height override: `height`, `heightFrom`.
- Layout: `spacing`.
- Wash: `washVolume`, `washCycles`, `span8WashVolume`, `span8WasteVolume`.
- Technique name: `prototype` (may be a literal technique name *or* an expression that
  resolves to one at run time).
- Pod name: `pod` (a literal `"Pod1"`/`"Pod2"` or an expression resolving to one).
- Labware position: `where` / `location` (a literal deck position or an expression).

Per-step docs mark each key "expression-capable" where it applies; when in doubt, check the
step's parameter table.

## 5. What is *not* expression-capable

- **Structural / envelope fields** — `stepType`, `format`, `formatVersion`, and the
  step-node fields (`disabled`, the shape of `parameters`/`subSteps`). These are read as
  literals only.
- **Enumerated discriminators** read as fixed tokens — e.g. `stop` (`"Sources"` /
  `"Destinations"` / `"Either"`). The `liquidType` sentinels (`"Well Contents"` /
  `"Tip Contents"`) are also fixed tokens on the steps that use them; a few step families
  (Fixed-8 and Multichannel Select Tips, for example) do mark `liquidType` as
  expression-capable in their parameter tables — check the step's own table.
- **`_biomekType`** and the comArray shape keys (`arraySubtype`, `dimensionCount`,
  `lowerBound`).

If a field is not documented as expression-capable, author a literal.

## 6. Common mistakes

- **Forgetting the `=`.** `"dose * 2"` is a literal string, not a formula — it will be
  coerced (often to 0 for a numeric field) instead of evaluated.
- **Writing a bare number where an expression field wants a string.** `"amount": 10` may
  round-trip oddly; author `"amount": "10"`.
- **Setting the expression key but not the flag** (or vice versa). If `useWellExpression`
  is `false`, your `firstWellExpression` is dead weight; the engine reads `firstWell`.
- **Referencing a variable that is not in scope.** The name must be defined by an enclosing
  `let`/`weak`/`prompt`, a global, or a loop/worklist counter, or evaluation fails at run
  time.
- **Assuming a stride matches the probe count.** `"=1+(Col-1)*8"` is a classic error — see
  [Concept Guide: Probe & Mandrel Selection](03-probe-and-mandrel-selection.md). To sweep plate columns the correct
  expression is simply `"=Col"` (no stride multiplier at all), because with row-major
  numbering the top well of column `c` is well `3`. A `WellsX` stride is for iterating
  *rows*, not columns, and even then it is never the pod's probe count.
