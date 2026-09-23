# Named Container

| Property | Value |
|----------|-------|
| stepType | `"Named Container"` |
| Category | Free-form container |
| Terminator | `"End"` |
| Compatible Hardware | All |

## Behavior Summary

The Named Container step groups child steps under a caption and enqueues them in order. It adds no control flow — no condition evaluation, no looping, no variable binding — and it performs no enqueue-time validation or configuration.

Every Named Container is a branch of an `"If"` step. `"If"` is the only step
that uses Named Container. Do not add Named Container anywhere other than in
an `"If"` step's substeps.

## Parameters Reference Table

The Named Container step has no step-specific parameters of its own. Its one meaningful key is the base key `caption` — and unlike most base keys, you should author it, because JSON import does not supply it (see the note below the table).

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `caption` | `string` | — | No, but author it in practice | `""` | Branch label shown in the step tree and in printed methods. Not expression-evaluated. Use `"Then"` / `"Else"` for a plain If branch, or a descriptive label such as `"Wash the Tips"`. **Always author this key.** |

**Why author `caption` on a Named Container.** The `"Then"` / `"Else"` captions are written by the If step only when the *editor* creates the If. JSON import replaces the step's parameters wholesale, so `caption` is not filled in for you. A branch authored without `caption` therefore imports with no caption at all, and the editor falls back to the built-in label `"Named Container"` for both branches.

Preserve `helpContextID: 13` when round-tripping an existing method.

> _Other universal base keys (`dynamic?`, `stepUI`, `disabled`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

## Enumerated / Constrained Values

No enumerated constraints.

## Cross-Field Validation Rules

No cross-field validation rules. The Named Container performs no runtime validation; it enqueues its children in order.

## Structural Context

The Named Container is a **free-form container**. It requires one terminal child:

- **Trailing anchor**: `"End"` terminator.

**Constraints**:

- `subSteps` must be present and contain at least the `"End"` terminator as the last element. Omitting `subSteps` fails import.
- The last element must be `"End"`. Any other type fails import.
- Any number of operational steps may be placed before the `"End"` terminator.
- A Named Container with only `"End"` (no operational steps) is valid — it represents an empty branch.

## Canonical Examples

Empty Named Container (an empty Else branch):

```json
{
  "stepType": "Named Container",
  "parameters": { "caption": "Else" },
  "subSteps": [
    { "stepType": "End", "parameters": {} }
  ]
}
```

Named Container with operational steps (a Then branch):

```json
{
  "stepType": "Named Container",
  "parameters": { "caption": "Then" },
  "subSteps": [
    { "stepType": "Transfer", "parameters": {} },
    { "stepType": "Move Labware", "parameters": {} },
    { "stepType": "End", "parameters": {} }
  ]
}
```

Named Container with a descriptive branch label (common in real methods) and a nested container:

```json
{
  "stepType": "Named Container",
  "parameters": { "caption": "Wash the Tips" },
  "subSteps": [
    {
      "stepType": "Group",
      "parameters": { "description": "Wash cycle" },
      "subSteps": [
        { "stepType": "Transfer", "parameters": {} },
        { "stepType": "End", "parameters": {} }
      ]
    },
    { "stepType": "End", "parameters": {} }
  ]
}
```

## Common Mistakes

- **Forgetting the `"End"` terminator**: Named Container is a free-form container and must end with `"End"`.
- **Omitting `caption` on an If branch**: JSON import replaces the step's parameters wholesale, so the parent If does not supply `"Then"` / `"Else"`. Author `caption` on each branch — `"Then"` / `"Else"`, or a descriptive label such as `"Wash the Tips"` (both are common in real methods). With no caption the editor labels the branch `"Named Container"`.
- **Using Named Container outside an `"If"`**: structural validation does not reject a Named Container at method root level between Start and Finish, but the editor never creates one there and the imported node is read-only, so it cannot be deleted afterwards. Use `"Group"` when you want a plain captioned container.
