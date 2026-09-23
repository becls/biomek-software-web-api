# Deck Layout: Biomek i3

- Source: `deckLayouts.i3.positions` in the Biomek Instrument Settings export.
- Illustrative view: `$id`/`$ref` references are resolved inline, and hardware-internal keys (ALP `group` blocks, footprint spans, framing internals, Pod-2 coordinates, geometry-only fields, runtime state) are stripped. Only the per-position keys you need to read a deck remain.
- Pod2 note: Pod2 coordinates (`x2`/`y2`/`z2`/`framed2` in the raw export) are omitted from the illustrative view. The i3 has a single Fixed-8 pod, so Pod2 is a legacy placeholder — not a second physical pod.
- Full key reference: see [Deck Layouts Format Specification](../deck-layouts-format-spec.md).
- ALP types: see [ALP Catalog](../catalog-alps.md).
- **No trash position.** This deck (`P1`--`P12`, `S1`--`S3`) carries no tip-discard position, so
  `"<Any Trash>"` cannot resolve on it — return tips with `"<where they came from>"` instead. See
  [Instrument Setup](../../../step-documentation/03-Instrument-Setup.md) for the full rules.

> **Illustrative, not a literal export.** This is a cleaned, readability-oriented view of the deck: `$id`/`$ref` references are resolved inline and non-position detail (ALP hardware groups, framing/geometry internals, Pod-2 coordinates, ...) is removed. In particular, every position in your real export carries a `group` object (a *position group*) — the shared definition of the ALP that position belongs to, linked across its positions via `$id`/`$ref`; it is per-ALP hardware data, not position placement, so it is dropped here. Your real export looks messier — it uses `$id`/`$ref` object sharing and carries additional keys — so treat this as a guide to the keys, not a byte-for-byte copy or deterministic parser output. See the deck-layout format spec for the full key reference.

## Visual Layout

The grid below is a **derived aid** — it is not part of the export. It mirrors the Biomek **Deck Editor**'s layout of this deck. The deck is a grid of mounting holes addressed by **column letter** and **row number**, with the origin at the back-left corner: column `A` is the left edge and row 1 is the back of the deck, so the top row here is the back and the bottom row is the front (operator side). The Deck Editor does not label every hole — it labels one column per **deck plate** (hence `A`, then `F`, `M`, `T`, ... on an i5/i7, or `A`, `H`, `O`, `V` on an i3) and one row every 5 holes — and the grid below uses exactly those labeled columns and rows. Each ALP occupies one cell; an ALP whose body genuinely covers more than one deck plate (a trash slide, which overhangs the side of the deck) is drawn as one wide cell spanning them. The real export carries the cm coordinates (`x1`/`y1`) plus each ALP group's `column`/`row`.

```text
Row  A         H         O         V
5    [empty]   [P1]      [P6]      [P11]
10   [empty]   [P2]      [P7]      [S1]
15   [empty]   [P3]      [P8]      [S2]
20   [empty]   [P4]      [P9]      [S3]
25   [empty]   [P5]      [P10]     [P12]
```

## Deck (illustrative, resolved)

The block below mirrors the shape of `deckLayouts.i3` in the export, with `$ref` pointers resolved and non-load-bearing keys removed. Values shown (coordinates in cm, `alpType`, labware `class` / `properties.liquidtype` / `volumeType`, device `name`, characteristic and required keys) are copied verbatim from the export; the surrounding `labware` object is reduced to those fields, and `device` is collapsed from the export's device object to just its `name` string.

```json
{
  "name": "i3",
  "isDefault": true,
  "isReadOnly": false,
  "positions": {
    "p1": {
      "alpType": "Static1x1_i3",
      "name": "P1",
      "x1": 17.695800001907347,
      "y1": 5.5928997055053715,
      "z1": 8.6424,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "fixed-8 Can Load Tips": null
      }
    },
    "p10": {
      "alpType": "Static1x1_i3",
      "name": "P10",
      "x1": 32.27540000190735,
      "y1": 47.24889970550537,
      "z1": 8.6424,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "fixed-8 Can Load Tips": null
      }
    },
    "p11": {
      "alpType": "Static1x1_i3",
      "name": "P11",
      "x1": 46.85500000190736,
      "y1": 5.5928997055053715,
      "z1": 8.6424,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "fixed-8 Can Load Tips": null
      }
    },
    "p12": {
      "alpType": "Static1x1_i3",
      "name": "P12",
      "x1": 46.85500000190736,
      "y1": 47.24889970550537,
      "z1": 8.6424,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "fixed-8 Can Load Tips": null
      }
    },
    "p2": {
      "alpType": "Static1x1_i3",
      "name": "P2",
      "x1": 17.695800001907347,
      "y1": 16.006899705505372,
      "z1": 8.6424,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "fixed-8 Can Load Tips": null
      }
    },
    "p3": {
      "alpType": "Static1x1_i3",
      "name": "P3",
      "x1": 17.695800001907347,
      "y1": 26.420899705505374,
      "z1": 8.6424,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "fixed-8 Can Load Tips": null
      }
    },
    "p4": {
      "alpType": "Static1x1_i3",
      "name": "P4",
      "x1": 17.695800001907347,
      "y1": 36.83489970550538,
      "z1": 8.6424,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "fixed-8 Can Load Tips": null
      }
    },
    "p5": {
      "alpType": "Static1x1_i3",
      "name": "P5",
      "x1": 17.695800001907347,
      "y1": 47.24889970550537,
      "z1": 8.6424,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "fixed-8 Can Load Tips": null
      }
    },
    "p6": {
      "alpType": "Static1x1_i3",
      "name": "P6",
      "x1": 32.27540000190735,
      "y1": 5.5928997055053715,
      "z1": 8.6424,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "fixed-8 Can Load Tips": null
      }
    },
    "p7": {
      "alpType": "Static1x1_i3",
      "name": "P7",
      "x1": 32.27540000190735,
      "y1": 16.006899705505372,
      "z1": 8.6424,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "fixed-8 Can Load Tips": null
      }
    },
    "p8": {
      "alpType": "Static1x1_i3",
      "name": "P8",
      "x1": 32.27540000190735,
      "y1": 26.420899705505374,
      "z1": 8.6424,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "fixed-8 Can Load Tips": null
      }
    },
    "p9": {
      "alpType": "Static1x1_i3",
      "name": "P9",
      "x1": 32.27540000190735,
      "y1": 36.83489970550538,
      "z1": 8.6424,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "fixed-8 Can Load Tips": null
      }
    },
    "s1": {
      "alpType": "Short1x1_i3",
      "name": "S1",
      "x1": 46.85500000190736,
      "y1": 16.006899705505372,
      "z1": 2.8131,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "disallow ManualTeach": null,
        "fixed-8 Can Load Tips": null
      }
    },
    "s2": {
      "alpType": "Short1x1_i3",
      "name": "S2",
      "x1": 46.85500000190736,
      "y1": 26.420899705505374,
      "z1": 2.8131,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "disallow ManualTeach": null,
        "fixed-8 Can Load Tips": null
      }
    },
    "s3": {
      "alpType": "Short1x1_i3",
      "name": "S3",
      "x1": 46.85500000190736,
      "y1": 36.83489970550538,
      "z1": 2.8131,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "disallow ManualTeach": null,
        "fixed-8 Can Load Tips": null
      }
    }
  }
}
```
