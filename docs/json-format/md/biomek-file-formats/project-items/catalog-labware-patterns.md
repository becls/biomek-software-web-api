# Catalog: Labware Patterns

> These are the well-selection patterns (**Well Patterns**) included in Beckman's default projects.
> Biomek has no standalone JSON export for project items today, but you do not need one to *use* a
> default pattern: a step references it by name (see **Using a pattern in a step** below). The
> `hardwareCompatibility` note is **pod-based** (Multichannel / Span-8 / Fixed-8); chassis size does
> not affect it, and a hybrid instrument supports the union of its pods. It is a catalog annotation
> derived from the three single-pod default projects: each pod tag is derived from the
> single-pod default project that uses that pod (`Fixed-8` from `Biomek_i3Project`, `Multichannel`
> from `Biomek_i5Project-MC`, `Span-8` from `Biomek_i5Project-Span`); a pattern's compatibility is
> the set of those pods whose project includes it, or `["any"]` when all three do. The
> `Biomek_i7Project` (Hybrid) is not used to add pod tags.

## The default patterns

All four default patterns are **quadrant interleave selections** on a 24×16 (384-well) plate. Each
one picks the 96 wells that a single 96-well plate maps to when four 96-well plates are condensed
into one 384-well plate. They differ only in which quadrant they select:

| Pattern | Selects (24 columns × 16 rows, 1-indexed) |
|---|---|
| `UL_quad_384` | Upper-left interleave — the 96 wells at **odd rows × odd columns** |
| `UR_quad_384` | Upper-right interleave — **odd rows × even columns** |
| `LL_quad_384` | Lower-left interleave — **even rows × odd columns** |
| `LR_quad_384` | Lower-right interleave — **even rows × even columns** |

**Hardware compatibility:** all four are `["Multichannel", "Span-8"]`. They are included in the
`Biomek_i5Project-MC` and `Biomek_i5Project-Span` single-pod default projects; the Fixed-8
default project (`Biomek_i3Project`) carries no patterns. (The Hybrid `Biomek_i7Project` also
carries them, but it is not used to add pod tags.)

## Using a pattern in a step

Steps that take a well selection (for example Transfer) let you either embed a selection inline or
reference one of these default patterns **by name**. To use a default pattern, set `localPattern` to
`false` and name it in `referencedPattern`:

```json
{
  "localPattern": false,
  "referencedPattern": "UL_quad_384"
}
```

With `localPattern` set to `false`, the step resolves the named pattern from the project's Well
Patterns, so you do not spell out the 384-well mask yourself. (On export the editor also emits empty
`pattern` and `selectionInfo` arrays alongside these keys for round-tripping; they are ignored when
a pattern is referenced by name.) To supply your own selection instead, set `localPattern` to `true`
and provide the mask inline — see the Transfer and Define Pattern step documentation for that form.
