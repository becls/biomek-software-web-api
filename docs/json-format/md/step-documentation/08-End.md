# End

| Property | Value |
|----------|-------|
| stepType | `"End"` |
| Category | Boundary (terminator) |
| Terminator | N/A — it IS the terminator |
| Compatible Hardware | All (no hardware restriction — inherits default) |

---

## Behavior Summary

The End step is a purely structural marker that terminates every standard container step in a Biomek method. It performs no action at runtime. Its sole purpose is to serve as the required last child of container steps such as Group, Loop, Let, Worklist, Scripted Let, Just In Time, Hold Labware, Named Container (used by If), and Define Procedure. The one container family that does **not** use End is Multichannel Select Tips, which instead uses the specialized `"Multichannel Select Tips End"` terminator.

---

## Parameters Reference Table

The End step reads nothing from the step's parameters. However, parent containers write keys to the End step's parameters, and these keys appear in serialized JSON. They should be preserved for round-trip fidelity but never modified when authoring or editing a method by hand.

The End step has no step-specific parameters. Universal base keys written by the parent container are `caption` and `helpContextID`; other base keys (`dynamic?`, `stepUI`, …) are defined in [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md) but are not written to End step parameters.

**Note**: When constructing a method from scratch, either preserve whatever caption the parent container sets (see the Structural Context section table) or omit base keys entirely.

---

## Enumerated / Constrained Values

None. The End step has no parameters and therefore no enumerated or constrained values.

---

## Cross-Field Validation Rules

None. Because the End step carries no step-specific dictionary keys, there are no cross-field dependencies to validate.

---

## Structural Context

The End step is the **default container terminator** in the Biomek step tree.

### Containers That Require End as Their Last Child

Every container in this table must have an `"End"` step as the final element of its `subSteps` array (structural rule). The override caption shown is the display label set by each container when it creates the End step.

| Container stepType | Override Caption |
|--------------------|-----------------|
| `"Group"` | `"End Group"` |
| `"Loop"` | `"End Loop"` |
| `"Let"` | `"End Let"` |
| `"Worklist"` | `"End Worklist"` |
| `"Scripted Let"` | `"End Scripted Let"` |
| `"Just In Time"` | `"End Just In Time"` |
| `"Hold Labware"` | `"End Holding Labware"` |
| `"Define Procedure"` | `"End Procedure"` |
| `"Named Container"` | Inherits default `"End"` |

### Named Container and If

The `"If"` step itself does not directly contain an End step. Instead, it contains exactly two `"Named Container"` children (Then and Else branches), and **each** of those Named Container children must have an `"End"` as its last child.

### The Exception: Multichannel Select Tips

`"Multichannel Select Tips"` does **not** use the generic End step. It uses `"Multichannel Select Tips End"` as its terminator instead. This is the only container family with a specialized terminator.

### Structural Rules

- A bottom-anchor step (`End`) must be at the last index in its sibling list.
- Every container in the fixed-terminator table must have its expected terminator as the last child.
- The End step is a leaf — it must never have a `subSteps` array with elements.

---

## Canonical Examples

### End as Last Child of a Group

```json
{
  "stepType": "Group",
  "parameters": {},
  "subSteps": [
    { "stepType": "Span-8 Aspirate", "parameters": {} },
    { "stepType": "Span-8 Dispense", "parameters": {} },
    { "stepType": "End", "parameters": { "caption": "End Group" } }
  ]
}
```

### End as Last Child of a Loop

```json
{
  "stepType": "Loop",
  "parameters": {},
  "subSteps": [
    { "stepType": "Transfer", "parameters": {} },
    { "stepType": "End", "parameters": { "caption": "End Loop" } }
  ]
}
```

### End in an If Step (Both Branches)

Each Named Container branch requires its own End step:

```json
{
  "stepType": "If",
  "parameters": {},
  "subSteps": [
    {
      "stepType": "Named Container",
      "parameters": { "caption": "Then" },
      "subSteps": [
        { "stepType": "Transfer", "parameters": {} },
        { "stepType": "End", "parameters": {} }
      ]
    },
    {
      "stepType": "Named Container",
      "parameters": { "caption": "Else" },
      "subSteps": [
        { "stepType": "Pause", "parameters": {} },
        { "stepType": "End", "parameters": {} }
      ]
    }
  ]
}
```

### End in a Nested Container (Loop Inside Group)

End is required at every level of nesting:

```json
{
  "stepType": "Group",
  "parameters": {},
  "subSteps": [
    {
      "stepType": "Loop",
      "parameters": {},
      "subSteps": [
        { "stepType": "Transfer", "parameters": {} },
        { "stepType": "End", "parameters": { "caption": "End Loop" } }
      ]
    },
    { "stepType": "End", "parameters": { "caption": "End Group" } }
  ]
}
```

### Empty Container (End as Only Child)

A newly created container with no body steps yet — the End step is the sole child:

```json
{
  "stepType": "Let",
  "parameters": {},
  "subSteps": [
    { "stepType": "End", "parameters": { "caption": "End Let" } }
  ]
}
```

---

## Common Mistakes

- **Using the override caption as the `stepType`** — the caption (`"End Group"`, `"End Loop"`) is display metadata set by the container; `stepType` is always `"End"`.
- **Using generic `"End"` inside Multichannel Select Tips** — that container's terminator is `"Multichannel Select Tips End"`, not the standard End.
- **Forgetting End in one branch of an If** — Then and Else are independent Named Containers; each needs its own terminator.
- **Confusing End with Finish** — End terminates containers; the method root uses `"Finish"`. They are not interchangeable.
- **Placing body steps after End** — anything after End is outside the container as far as insert/move logic is concerned.
