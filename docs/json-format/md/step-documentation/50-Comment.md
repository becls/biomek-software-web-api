# Comment

| Property | Value |
|----------|-------|
| stepType | `"Comment"` |
| Category | Leaf |
| Terminator | N/A |
| Compatible Hardware | All |

## Behavior Summary

The Comment step is a no-op at runtime — it does not generate any instrument commands. It carries `description` and `comment` text shown in the method editor and printed in method reports. Use it to annotate method logic or leave notes for other method authors.

## Parameters Reference Table

| Key | Type | Units | Required | Default | Description |
|-----|------|-------|----------|---------|-------------|
| `description` | `string` | — | No | `""` | Short description shown in the step tree. Every editor-produced step carries this key. Not expression-evaluated. **Always author this key.** |
| `comment` | `string` | — | No | `""` | Longer comment text. Editor-written; not read at runtime. Not expression-evaluated. |

> _Universal base keys (`caption`, `dynamic?`, `stepUI`, …) apply to this step but are omitted here — see [Base Keys & Serialization Conventions](00-Base-Keys-and-Conventions.md)._

### Inactive Keys — not read at runtime (may appear in existing methods)

If any additional keys appear on a `Comment` step in an exported method, they do not impact the behavior of the Comment step. Preserve them verbatim for round-tripping.

## Enumerated / Constrained Values

None.

## Cross-Field Validation Rules

None. Both parameters are optional free-text strings.

## Structural Context

This step is a leaf and does not support subSteps.

## Canonical Examples

Simple comment:

```json
{
  "stepType": "Comment",
  "parameters": {
    "description": "Transfer reagent to plate",
    "comment": "Using 200µL tips for this transfer to avoid cross-contamination."
  }
}
```

## Common Mistakes

- **Expecting runtime behavior**: Comment does nothing at runtime. Do not use it as a substitute for actual logic.
- **Using expressions in description/comment**: Expressions in these fields will not be evaluated; text is stored as-is.
