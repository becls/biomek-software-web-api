# Deck Layout: Biomek i5 Span8

- Source: `deckLayouts.span8.positions` in the Biomek Instrument Settings export.
- Illustrative view: `$id`/`$ref` references are resolved inline, and hardware-internal keys (ALP `group` blocks, footprint spans, framing internals, Pod-2 coordinates, geometry-only fields, runtime state) are stripped. Only the per-position keys you need to read a deck remain.
- Pod2 note: Pod2 coordinates (`x2`/`y2`/`z2`/`framed2` in the raw export) are omitted from the illustrative view. On stock decks they equal Pod1 because the deck is unframed (uncalibrated); on a framed deck they carry the Pod2 calibration.
- Full key reference: see [Deck Layouts Format Specification](../deck-layouts-format-spec.md).
- ALP types: see [ALP Catalog](../catalog-alps.md).
- Duplicate-deck note: this configuration also exposes a `Standard` deck whose positions, coordinates, groups, and characteristics are byte-identical to this `Span8` deck (only `deckName` differs). No separate view is kept for it — this file is authoritative for both.

> **Illustrative, not a literal export.** This is a cleaned, readability-oriented view of the deck: `$id`/`$ref` references are resolved inline and non-position detail (ALP hardware groups, framing/geometry internals, Pod-2 coordinates, ...) is removed. In particular, every position in your real export carries a `group` object (a *position group*) — the shared definition of the ALP that position belongs to, linked across its positions via `$id`/`$ref`; it is per-ALP hardware data, not position placement, so it is dropped here. Your real export looks messier — it uses `$id`/`$ref` object sharing and carries additional keys — so treat this as a guide to the keys, not a byte-for-byte copy or deterministic parser output. See the deck-layout format spec for the full key reference.

## Visual Layout

The grid below is a **derived aid** — it is not part of the export. It mirrors the Biomek **Deck Editor**'s layout of this deck. The deck is a grid of mounting holes addressed by **column letter** and **row number**, with the origin at the back-left corner: column `A` is the left edge and row 1 is the back of the deck, so the top row here is the back and the bottom row is the front (operator side). The Deck Editor does not label every hole — it labels one column per **deck plate** (hence `A`, then `F`, `M`, `T`, ... on an i5/i7, or `A`, `H`, `O`, `V` on an i3) and one row every 5 holes — and the grid below uses exactly those labeled columns and rows. Each ALP occupies one cell; an ALP whose body genuinely covers more than one deck plate (a trash slide, which overhangs the side of the deck) is drawn as one wide cell spanning them. The real export carries the cm coordinates (`x1`/`y1`) plus each ALP group's `column`/`row`.

```text
Row  A         F         M         T         AA        AH        AO
5    [empty]   [empty]   [empty]   [empty]   [empty]   [empty]   [empty]
10   [empty]   [P1]      [P6]      [P11]     [P16]     [empty]   [W1]
15   [empty]   [P2]      [P7]      [P12]     [P17]     [TR1              ]
20   [empty]   [P3]      [P8]      [P13]     [P18]     [empty]   [empty]
25   [empty]   [P4]      [P9]      [P14]     [P19]     [empty]   [empty]
30   [empty]   [P5]      [P10]     [P15]     [P20]     [empty]   [empty]
```

## Deck (illustrative, resolved)

The block below mirrors the shape of `deckLayouts.span8` in the export, with `$ref` pointers resolved and non-load-bearing keys removed. Values shown (coordinates in cm, `alpType`, labware `class` / `properties.liquidtype` / `volumeType`, device `name`, characteristic and required keys) are copied verbatim from the export; the surrounding `labware` object is reduced to those fields, and `device` is collapsed from the export's device object to just its `name` string.

```json
{
  "name": "Span8",
  "isDefault": true,
  "isReadOnly": false,
  "positions": {
    "p1": {
      "alpType": "Static1x1",
      "name": "P1",
      "x1": 11.23100000190735,
      "y1": 15.547999705505372,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p10": {
      "alpType": "Static1x3",
      "name": "P10",
      "x1": 25.81060000190735,
      "y1": 57.20399963684082,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p11": {
      "alpType": "Static1x1",
      "name": "P11",
      "x1": 40.39020000190736,
      "y1": 15.547999705505372,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p12": {
      "alpType": "Static1x1",
      "name": "P12",
      "x1": 40.39020000190736,
      "y1": 25.961999705505374,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p13": {
      "alpType": "Static1x3",
      "name": "P13",
      "x1": 40.39020000190736,
      "y1": 36.375999636840824,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p14": {
      "alpType": "Static1x3",
      "name": "P14",
      "x1": 40.39020000190736,
      "y1": 46.78999963684082,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p15": {
      "alpType": "Static1x3",
      "name": "P15",
      "x1": 40.39020000190736,
      "y1": 57.20399963684082,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p16": {
      "alpType": "Static1x1",
      "name": "P16",
      "x1": 54.96980000190736,
      "y1": 15.547999705505372,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p17": {
      "alpType": "Static1x1",
      "name": "P17",
      "x1": 54.96980000190736,
      "y1": 25.961999705505374,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p18": {
      "alpType": "Static1x3",
      "name": "P18",
      "x1": 54.96980000190736,
      "y1": 36.375999636840824,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p19": {
      "alpType": "Static1x3",
      "name": "P19",
      "x1": 54.96980000190736,
      "y1": 46.78999963684082,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p2": {
      "alpType": "Static1x1",
      "name": "P2",
      "x1": 11.23100000190735,
      "y1": 25.961999705505374,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p20": {
      "alpType": "Static1x3",
      "name": "P20",
      "x1": 54.96980000190736,
      "y1": 57.20399963684082,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p3": {
      "alpType": "Static1x3",
      "name": "P3",
      "x1": 11.23100000190735,
      "y1": 36.375999636840824,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p4": {
      "alpType": "Static1x3",
      "name": "P4",
      "x1": 11.23100000190735,
      "y1": 46.78999963684082,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p5": {
      "alpType": "Static1x3",
      "name": "P5",
      "x1": 11.23100000190735,
      "y1": 57.20399963684082,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p6": {
      "alpType": "Static1x1",
      "name": "P6",
      "x1": 25.81060000190735,
      "y1": 15.547999705505372,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p7": {
      "alpType": "Static1x1",
      "name": "P7",
      "x1": 25.81060000190735,
      "y1": 25.961999705505374,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p8": {
      "alpType": "Static1x3",
      "name": "P8",
      "x1": 25.81060000190735,
      "y1": 36.375999636840824,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "p9": {
      "alpType": "Static1x3",
      "name": "P9",
      "x1": 25.81060000190735,
      "y1": 46.78999963684082,
      "z1": 15.875,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "span Can Load Tips": null
      }
    },
    "tR1": {
      "alpType": "TrashRightSlide",
      "name": "TR1",
      "x1": 69.8573999909401,
      "y1": 30.147900088500975,
      "z1": 20.612,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "standard TiterPlate Size": null
      },
      "characteristics": {
        "can Discard Tips": null,
        "disableFrameWithGripper": null,
        "disallow AutoTeach": null,
        "disallow ManualTeach": null,
        "disallow NormalTeach": null,
        "trash": null
      },
      "device": "Trash"
    },
    "w1": {
      "alpType": "WashStationSpan8",
      "name": "W1",
      "x1": 83.92169992294312,
      "y1": 15.342999946594238,
      "z1": 22.1361,
      "framed1": "Not",
      "permanentLabware": false,
      "required": {
        "no plates allowed": null
      },
      "characteristics": {
        "disableFrameWithGripper": null,
        "disallow AutoTeach": null,
        "disallow NormalTeach": null,
        "no Gripping Allowed": null,
        "vWash Station": null
      }
    }
  }
}
```
