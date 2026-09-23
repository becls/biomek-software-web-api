# Catalog: Tip Classes

> **The names here are real and usable.** These entries are the **default project items included
> with Biomek**: a method references them **by name**, and these are the valid names to use. Project
> items are not a file you author or import. A specific project may add its own, so treat this as
> the default set, not an exhaustive list.
>
> Every unique Beckman default tip class is listed once, deduplicated across the three single-pod
> default projects. The **Pods** column records `hardwareCompatibility`, which is **pod-based**
> (Multichannel / Span-8 / Fixed-8); chassis size does not affect it, and a hybrid instrument
> supports the union of its pods. The field is tentative: it is a catalog annotation derived from
> which default projects include the item (mapped to their pods), not necessarily an intrinsic
> field of the forthcoming format. A Pods value of `any` means all three default projects include the
> class. **It is a membership hint, not a capability guarantee** — it does not promise a given pod
> can reach the position you place the labware at.
>
> **Note on `LLS`.** The per-tip-class `LLS` boolean records whether the tip itself is
> LLS-capable. **Liquid-level sensing is only supported by the Span-8 pod**; Fixed-8 and
> Multichannel pods do not perform LLS regardless of the tip's `LLS` value. An LLS-capable
> tip class showing Pods `any` means the tip class is included in all
> three single-pod default projects — not that LLS itself works across all pods.
>
> **Operational consequence.** A non-LLS tip class (`LLS: false`, e.g. `T190F`, `T50F`)
> cannot be paired with an LLS-enabled Span-8 technique — the stock auto-selected Span-8
> techniques set `c__LLSInitial: true`, so with non-LLS tips the enqueue fails with
> `Cannot liquid level sense with tip type <class>.` To fix, either pick an LLS-capable tip
> class (`_LLS` variant), select a technique whose `c__LLSInitial` is `false`, or override
> height on the pipetting step (`overrideHeight: true` with `heightFrom` set to `Bottom` or
> `Top`).

## Scope and derivation

- **Category:** Tip Classes (`Tip Classes`).
- **Sources:** the three single-pod default projects (`Biomek_i3Project`,
  `Biomek_i5Project-MC`, `Biomek_i5Project-Span`).
- **Pod mapping used to derive `hardwareCompatibility`:** each pod tag is derived from the
  single-pod default project that uses that pod (`Fixed-8` from `Biomek_i3Project`,
  `Multichannel` from `Biomek_i5Project-MC`, `Span-8` from `Biomek_i5Project-Span`); a tip's
  compatibility is the set of those pods whose project includes it, or `["any"]` when all three
  do. The `Biomek_i7Project` (Hybrid) is not used to add pod tags — it carries both pods, so
  membership there cannot distinguish which pod a tip is for. Excluding it costs no coverage: its
  26 tip classes are exactly the 26 in the union of the three single-pod projects.

## Naming scheme

Disposable tip class names decode as `T<capacity>[F][_LLS | _WB][_384]`:

| Element | Meaning |
|---|---|
| `T<number>` | Nominal capacity in µL. `T190F` holds 190 µL; `T1070` holds 1070 µL. Capacity is inclusive of liquid and trailing air gap. |
| `F` | Filtered (aerosol-resistant). Present exactly when `Filtered: true`. |
| `_LLS` | LLS-capable variant. Present exactly when `LLS: true` on a disposable tip. |
| `_WB` | Wide-bore variant, intended for viscous liquids and cell suspensions. |
| `_384` | 384-format tip for a 384-channel head on a Multichannel pod. |

Consequences of the scheme when picking a class by name:

- Filtered and unfiltered variants at the same tip length are separate classes with different
  capacities. `T190F` (190 µL) and `T230` (230 µL) share a tip height of 5.1435 cm; `T40F`/`T80`,
  `T50F`/`T90`, and `T1025F`/`T1070` pair the same way at their own lengths.
- A `_WB` class carries exactly the same published property values as its base class — capacity,
  filtering, LLS, tip height and air capacity are all identical. The difference is the orifice: a
  `_WB` tip narrows to a 0.0991 cm radius against 0.0635 cm on its base class. That geometry is not
  one of the properties below, so the two are told apart by name alone.
- `_LLS` and `_WB` variants exist only for 96-format tips. The four `_384` classes have neither.
- The two fixed Span-8 tips, `Fixed100` and `SeptaFluted`, do not follow this scheme. Both are
  `Fixed: true`, LLS-capable, and draw from system tubing; `SeptaFluted` is the piercing tip used
  with septa-capped tube racks.

## Items

Each of these tip classes has a matching `BC*` tip box; see
[Tip Boxes](catalog-labware-classes.md#tip-boxes) for the box names and the box-to-tip pairing.

All 24 disposable classes are unfixed, non-piercing, and do not draw from system tubing.
`Fixed100` and `SeptaFluted` are the two fixed Span-8 tips: both draw from system tubing and are
LLS-capable, and `SeptaFluted` is the only piercing tip in the set.

**Capacity** is the maximum liquid plus
*trailing* air gap the tip will hold, and excludes the leading air gap.

| Class | Capacity (µL) | Filtered | LLS | Tip height (cm) | Pods |
|---|---|---|---|---|---|
| `T1025F` | 1025.0 | Yes | No | 10.9855 | any |
| `T1025F_LLS` | 1025.0 | Yes | Yes | 10.9855 | any |
| `T1025F_WB` | 1025.0 | Yes | No | 10.9855 | any |
| `T1070` | 1070.0 | No | No | 10.9855 | any |
| `T1070_LLS` | 1070.0 | No | Yes | 10.9855 | any |
| `T1070_WB` | 1070.0 | No | No | 10.9855 | any |
| `T190F` | 190.0 | Yes | No | 5.1435 | any |
| `T190F_LLS` | 190.0 | Yes | Yes | 5.1435 | any |
| `T190F_WB` | 190.0 | Yes | No | 5.1435 | any |
| `T230` | 230.0 | No | No | 5.1435 | any |
| `T230_LLS` | 230.0 | No | Yes | 5.1435 | any |
| `T230_WB` | 230.0 | No | No | 5.1435 | any |
| `T40F` | 40.0 | Yes | No | 3.8252 | any |
| `T40F_LLS` | 40.0 | Yes | Yes | 3.8252 | any |
| `T50F` | 50.0 | Yes | No | 5.1359 | any |
| `T50F_LLS` | 50.0 | Yes | Yes | 5.1359 | any |
| `T80` | 80.0 | No | No | 3.8252 | any |
| `T80_LLS` | 80.0 | No | Yes | 3.8252 | any |
| `T90` | 90.0 | No | No | 5.1359 | any |
| `T90_LLS` | 90.0 | No | Yes | 5.1359 | any |
| `T25F_384` | 25.0 | Yes | No | 3.3147 | Multichannel |
| `T30_384` | 30.0 | No | No | 3.3147 | Multichannel |
| `T40F_384` | 40.0 | Yes | No | 4.3104 | Multichannel |
| `T50_384` | 50.0 | No | No | 4.3104 | Multichannel |
| `Fixed100` | 93.0 (see note) | No | Yes | 10.185 | Span-8 |
| `SeptaFluted` | 37.0 (see note) | No | Yes | 11.938 | Span-8 |

Note: Fixed tips (`Fixed100`, `SeptaFluted`) enable drawing liquid into the
Span-8 pod's tubing. Depending on the installed tubing and syringe, the volume
varies. The given volumes are for large volume tubing, assuming no liquid is
drawn into the tubing. Refer to the Biomek i5 and Biomek i7 Instructions for
Use for more information (B54473).
