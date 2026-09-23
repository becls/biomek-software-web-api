# Concept Guide: The Variable Environment (`let` / `weak` / `prompt`)

A Biomek method runs against a **scope** of named variables. Any expression (`=name`)
resolves the name against the scope in effect at that step. Three JSON keys —
`let`, `weak`, and `prompt` — control how variables come into scope. This guide gathers
the rules that govern them in one place. Cross-link: `01-expressions.md` for the
expression language itself.

---

## 1. The three binding kinds

Each of the three keys is a JSON object (a dictionary) whose keys are variable names and
whose values are either literal strings/numbers or `=`-prefixed expressions:

```jsonc
"let":    { "SourceCol": "1",       "Volume": "=BaseVolume * 2" }
"weak":   { "TipType":   "P200" }
"prompt": { "OperatorName": null }
```

- **`let`** — a *strong* binding. Defines or overrides the variable for this step's
  descendants, shadowing any outer definition of the same name.
- **`weak`** — a *weak* binding. Takes effect only if the variable is **not** already
  defined in an enclosing scope. Use this for defaults that a caller can override.
- **`prompt`** — the variable's value is asked of the operator at run time. Only the
  **key** (the variable name) matters; the value stored in the `prompt` object is **not
  used** and should be authored as `null`. The dialog message is generated automatically
  from the variable name (`Enter a value to use for '<name>'`). Every prompted variable
  must **also** appear in `let` — that `let` entry supplies the default shown in the
  dialog. A `prompt` entry with no matching `let` key has no effect and produces no
  dialog.

Variable names should be valid script identifiers — no spaces, no leading digit.

## 2. Scoping

Bindings introduced by a scope step are visible to that step's descendants and go **out
of scope** when execution leaves the container. Strong (`let`) shadows any outer
definition of the same name; weak fills in only when the name is otherwise unset.

## 3. Which steps carry the environment keys

All three keys are present, and **required at enqueue**, on the container steps that open
a new scope: **Start, Let, Run Method, Define Procedure, Run Procedure**. Include them
even when empty — a bare `"let": {}, "weak": {}, "prompt": {}` is fine; omitting one of
these keys on a scope step hard-fails at Run.

**Scripted Let** is a partial exception. It requires `prompt` to be present (empty `{}` is fine)
— an omitted `prompt` hard-fails at Run just like on the other scope steps — but its `let` and
`weak` keys are reset internally before the script runs, so omitting `let` and `weak` is safe on
Scripted Let. See
[Scripted Let](../step-documentation/67-Scripted-Let.md) for the exact
per-key behavior.

Some exported **Script** steps carry a `let` (and `weak`) key left over from editor
state. It has **no runtime effect** on a Script step — the Script step ignores these keys
entirely — so do not author them there.

**Worklist does not persist these keys.** A Worklist step builds its per-row scope
internally; do not author `let` / `weak` / `prompt` on a Worklist step.

## 4. Globals and other name sources

Scoped `let` / `weak` / `prompt` are only one of several ways a name becomes visible to
an expression. Distinguish them from:

- **Globals** — created by Set Global, Next Item, Create Labware Group, and Next Labware.
  Globals live for the whole method run and beyond rather than being tied to a container's scope.
- **Loop / Worklist counters** — a Loop or Worklist step exposes its counter variable to
  its children.
- **Pipetting `C__…` context variables** — a separate, engine-injected scope used inside
  a technique's template. Not authored here; see [Pipetting Techniques, Templates, and Auto-Selection](../Pipetting-Techniques-and-Templates.md).

All of these are names an expression can reference, but their lifetimes and authoring
mechanisms differ.

## 5. Author's short list

- On Start — and on every scope step that requires them — include `let` / `weak` /
  `prompt` even when their objects are empty.
- Use `let` for values you set directly, `weak` for overridable defaults, and `prompt`
  for values you want the operator to type at Run.
- For every `prompt` entry, add a matching `let` entry (its value is the dialog default)
  and author the `prompt` value as `null`.
- Reference the bound names from child steps with `=Name`.
