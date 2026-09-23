# Multichannel Select Tips End

> **Reference manual:** Chapter 20 documents these as *Select Tips* steps (the manual omits the *Multichannel* prefix); the JSON `stepType` keeps the `Multichannel Select Tips …` name.

| Property | Value |
|----------|-------|
| stepType | `"Multichannel Select Tips End"` |
| Category | Terminator (auto-generated) |
| Terminator | N/A (this IS the terminator) |
| Compatible Hardware | Any / not restricted |

---

## Behavior Summary

The `"Multichannel Select Tips End"` step (display caption `"End Using Select Tips"`) is the mandatory terminator for a `"Multichannel Select Tips"` container. It performs a single validation: the multichannel pod **must have no tips loaded** at this point. If tips are still on the pod, it throws an error. This ensures every Select Tips block unloads its tips before exiting — a Select Tips group that loaded tips must unload them (via a Select Tips Unload / Advanced Unload child) before the terminator.

This step cannot be copied, moved, or edited by the user via the user interface. When authoring a method via JSON, include it explicitly as the last element of the container's `subSteps` array.

---

## Parameters Reference Table

None. This step has no user-configurable parameters.

---

## Cross-Field Validation Rules

1. The multichannel pod **must have no tips loaded** when this step executes. If tips remain, it raises an exit-with-tips error. Unload tips inside the container before this terminator.

---

## Structural Context

This step is the **terminator** for `"Multichannel Select Tips"`. It must always be the last element in `subSteps`.

```json
{ "stepType": "Multichannel Select Tips End", "parameters": {} }
```

---

## Common Mistakes

- **Placing anything after the terminator inside `subSteps`**: the terminator must always be the final element in the container's `subSteps` array.
- **Manually editing this step**: it is auto-generated and read-only; do not add parameters or move it out of the container.
- **Reaching the terminator with tips still on the pod**: include a Select Tips Unload (or Advanced Unload) child before this step. See rule 1.

---

## Related Guides

- **[Multichannel Select Tips](26-Multichannel-Select-Tips.md)** — the container this step terminates; explains the tips-off → tips-off bracket and rearrange-position semantics.
