# Worked Example: Full i5 Span-8 Method

A whole method, start to finish, in the order it is built: a plate-absorbance QC run on an
i5 Span-8. It is longer than the [i3 Fixed-8 worked example](Worked-Example-End-to-End.md)
and is here for a different reason — that page shows the smallest thing that works, this one
shows what a real method looks like once a deck, a control read, a column sweep, and a
device that has no step of its own are all in the same file.

Everything below is one method. The full JSON is at the end of the page in a single block;
the sections before it quote pieces of that same block and explain why each key holds the
value it does.

---

## 1. What the method does

Quantify nucleic acid concentration across a 96-well sample plate by absorbance, then
normalize every sample to a common target concentration in a fresh output plate.

1. **Blank.** 30 µL of blank buffer goes from a reservoir into column 1 of a dedicated
   reader plate. The reader measures it and records the baseline optical density that is
   subtracted from every later reading.
2. **Sample load.** 25 µL from each of the 96 sample wells goes into the matching well of a
   second reader plate, one plate column per pass, fresh tips each pass. The reader measures
   concentration and the A260/A280 and A260/A230 purity ratios.
3. **Normalize.** 30 µL of each sample plus 30 µL of diluent go into the output plate, again
   one column per pass, with a tip change between the sample leg and the diluent leg.

The absorbance reader is a third-party device. A real method drives it with that device's own
add-on step; this example has no such step available, so Section 5 shows the placeholder pattern
used in its place — see the caveat there.

Two choices below drive several others:

For this baseline read, each optically-read well should hold a single known liquid so the
measurement isn't confounded. The blank and the samples therefore go into two separate plates. A
96-well plate has no spare well once 96 samples occupy it, so there is nowhere on the sample plate
to put a blank without overwriting a well that will be read.

The output plate is the one place two liquids are deliberately combined — sample plus
diluent. It is never read, so that is a mix, not a contamination.

## 2. Instrument and deck

Target: **i5 Span-8**, pod `Pod1`, disposable LLS-capable tips (`BC230_LLS` boxes holding
`T230_LLS` tips). Deck `Span8`.

| Position | Labware class | Instance name | `volumeType` | Seeded contents |
|----------|---------------|---------------|--------------|-----------------|
| P1, P2, P16, P17 | `BC230_LLS` | — | `Unknown` | tip boxes, 96 tips each |
| P3 | `BCFullReservoir` | `BlankReservoir` | `Known` | 1 section, 50 000 µL Water |
| P4 | `BCFullReservoir` | `DiluentReservoir` | `Known` | 1 section, 50 000 µL Water |
| P6 | `BCFlat96` | `SamplePlate` | `Known` | 96 wells, 200 µL Water each |
| P7 | `BCFlat96` | `CuvettePlate` | `Known` | 96 wells, 0.0 µL |
| P8 | `BCFlat96` | `BlankCuvettePlate` | `Known` | 96 wells, 0.0 µL |
| P11 | `BCFlat96` | `DestPlate` | `Known` | 96 wells, 0.0 µL |
| all others | — | — | — | declared empty (`[]`) |

Four tip boxes are on the deck because the method runs 37 tip-load cycles of 8 probes — 296
tips — and one box holds 96. Run out of tips and the load fails at run time with a
"unable to find and retrieve tips" error, so count them before you author.

`layout: "Span8"` names a **deck defined on that instrument**, not a pod family, and is the one value
on this page not to copy — deck names are per-instrument, and `"Span8"` may be absent on yours. Read
the name from your own settings export, or omit `layout` and take the instrument's default deck.

## 3. The envelope and Start

```jsonc
{
  "format": "Biomek Method",
  "formatVersion": "1.0",
  "author": "",
  "description": "Absorbance concentration check with normalization on i5 Span-8 (disposable tips)",
  "steps": [ ... ]
}
```

`format` and `formatVersion` are matched exactly on import — author the version shown above (the
method JSON format version, not a software release); import rejects an older value, and a longer
form such as `"1.0.0"` is rejected as newer. `author` and `description` are free
text and may be empty. `steps` is the method tree — a flat array here, except where a Loop
carries its children in `subSteps`.

```json
{
  "stepType": "Start",
  "parameters": {
    "bitmap": "OStepUI.ocx,START",
    "let": {},
    "weak": {},
    "prompt": {}
  }
}
```

`let`, `weak`, and `prompt` are the three variable dictionaries a method can declare at
Start. All three are empty here: the only variable this method uses is the Loop counter
`Col`, and a Loop exposes its counter to its own children — it does not have to be declared
at Start. Emit all three keys even when their objects are empty. See
[The Variable Environment](concept-guides/04-variable-environment.md).

## 4. Instrument Setup: declaring the deck

Instrument Setup is where this method places its labware, and every pipetting step afterwards
references labware placed here — a pipetting step never places a plate, so set the deck up first.
(A later Instrument Setup step can place more labware, and some add-on steps place labware too, but
keep to a single Instrument Setup where you can.)

A tip box:

```jsonc
"p1": [
  {
    "_biomekType": "labware",
    "class": "LabwareClasses\\BC230_LLS",
    "volumeType": "Unknown",
    "properties": {},
    "tipType": "TipClasses\\T230_LLS"
  }
],
```

- The `deckItems` **key** is the position name camelCased — `p1`, not `P1`. Pipetting steps
  later use the display form (`"P8"`). A key that matches no position on the selected deck
  is silently ignored, and the labware is simply never placed, so a typo here surfaces much
  later as a "position does not exist" or class-mismatch failure.
- `class` is **path-prefixed** here (`LabwareClasses\\BC230_LLS`). Every pipetting step below
  uses the **short** name (`"BCFlat96"`, `"BCFullReservoir"`). Swapping the two forms is a
  hard enqueue failure in both directions.
- A tip box holds no liquid, so `volumeType` is `Unknown` and there are no `evalAmounts`.
- `tipType` names the tip class the box dispenses. The box class and tip class pair by
  suffix: `BC230_LLS` → `T230_LLS`. Name a tip class the box does not actually hold and tip
  loading fails.

A reservoir:

```jsonc
"p3": [
  {
    "_biomekType": "labware",
    "class": "LabwareClasses\\BCFullReservoir",
    "volumeType": "Known",
    "properties": {
      "name": "BlankReservoir",
      "liquidtype": "Water"
    },
    "evalAmounts": {
      "_biomekType": "comArray",
      "arraySubtype": "variant",
      "values": [50000.0]
    },
    "evalLiquids": {
      "_biomekType": "comArray",
      "arraySubtype": "variant",
      "values": ["Water"]
    }
  }
],
```

- `evalAmounts` and `evalLiquids` must be **exactly as long as the labware's well count** —
  or, for a reservoir, its **section** count. `BCFullReservoir` is a single 200 000 µL
  section, so both arrays hold one element. Writing 96 entries here because other labware on
  the deck have 96 wells is wrong and fails — a reservoir's section count is not a plate's
  well count.
- `properties.name` gives the labware an instance name. Downstream position fields resolve a
  deck position, a labware instance name, or a labware class name interchangeably, which is
  why the aspirate steps below can say `"where": "BlankReservoir"` instead of `"P3"`. Names
  survive a deck rearrangement; positions do not.

A plate seeded with sample, and an empty destination plate (abridged — the real arrays hold
96 entries each):

```jsonc
"p6": [
  {
    "_biomekType": "labware",
    "class": "LabwareClasses\\BCFlat96",
    "volumeType": "Known",
    "properties": { "name": "SamplePlate" },
    "evalAmounts": { "_biomekType": "comArray", "arraySubtype": "variant",
                     "values": [200.0, 200.0, "… 96 entries …"] },
    "evalLiquids": { "_biomekType": "comArray", "arraySubtype": "variant",
                     "values": ["Water", "Water", "… 96 entries …"] }
  }
],
```

The three destination plates (P7, P8, P11) are declared the **same way**, with `Known` and
96 amounts of `0.0`. That looks redundant for an empty plate and is the single most common
reason a first method fails: the default techniques position the tip relative to the liquid
surface, and a destination declared `Unknown` has no locatable surface before anything has
been dispensed into it. The first dispense then fails with *"Cannot move relative to liquid
level if liquid amounts are not defined for specified labware."* Declare empty destinations
`Known` (or `Nominal`) with an all-zero `evalAmounts` and a full-length `evalLiquids`.

Positions the method does not use are still listed, as `[]`, which clears them.

The step-level keys:

```jsonc
"layout": "Span8",
"pause?": true,
"podSetup": {
  "leftHasTips": false,
  "leftTipType": "",
  "rightHasTips": false,
  "rightTipType": ""
},
"verifyPodSetup?": true
```

`pause?: true` shows the setup-confirmation dialog so the operator can check the deck against
what the method declared. `verifyPodSetup?` plus `podSetup` assert that neither pod side
starts with tips on it — this sets the enqueue-time model, it does not physically move tips.
If the pod really does start loaded and the method says otherwise, the first Load Tips step
discards whatever is there.

## 5. The blank leg

Load tips, aspirate 30 µL of blank from the reservoir, dispense it into column 1 of
`BlankCuvettePlate`, hand the plate to the reader, unload tips.

```json
{
  "stepType": "Span-8 Load Tips",
  "parameters": {
    "pod": "Pod1",
    "tips": "BC230_LLS",
    "useProbes": {
      "_biomekType": "comArray",
      "arraySubtype": "boolean",
      "values": [true, true, true, true, true, true, true, true]
    },
    "useExpression": false
  }
}
```

`tips` is a multi-purpose locator: a tip **class** (as here), a box **instance name**, or a
**deck position**. Because no position is named, the step finds any reachable box of that
class — which is why four interchangeable boxes on P1/P2/P16/P17 work with no further
bookkeeping. `useExpression: false` makes `useProbes` authoritative; set it `true` and
probe selection comes from `mandrelExpression` instead and `useProbes` is ignored.

```json
{
  "stepType": "Span-8 Aspirate",
  "parameters": {
    "amount": "30",
    "aspirate": true,
    "autoSelectPrototype": true,
    "prototype": "",
    "firstWell": 1,
    "firstWellExpression": "",
    "useWellExpression": false,
    "spacing": "",
    "height": 2.0,
    "heightFrom": 1,
    "overrideHeight": true,
    "customHeight": false,
    "liquidtype": "Well Contents",
    "operation": "Aspirate",
    "pod": "Pod1",
    "what": "BCFullReservoir",
    "where": "BlankReservoir",
    "useProbes": { "_biomekType": "comArray", "arraySubtype": "boolean",
                   "values": [true, true, true, true, true, true, true, true] },
    "discardExcess": false,
    "emptyTips": false,
    "mandrelExpression": "",
    "refreshTips": false,
    "tipLabwareClass": "",
    "useExpression": false
  }
}
```

- `where` names the reservoir by its instance name; `what` is the **short** class name.
  Both must agree with what Instrument Setup placed at that position, or enqueue fails with a
  class mismatch.
- `spacing: ""` — a probe-to-probe stride is meaningless on a one-section reservoir. Empty
  makes the step substitute the labware's own minimum interval. All eight probes draw from
  the single section.
- `overrideHeight: true` with `height: 2.0` and `heightFrom: 1` puts the tips **2 mm off the
  section bottom**. `heightFrom` is `1` = bottom (`0` = liquid surface, `2` = well top). A
  fixed bottom reference is deliberate here: a deep trough is a poor target for a
  liquid-relative move. On the plates further down, `overrideHeight` is `false` and the
  technique's own height governs — where `overrideHeight` is `false`, the `height` and
  `heightFrom` values sitting next to it are inert.
- `liquidtype: "Well Contents"` takes the liquid type from what the labware is tracked as
  holding. `"Tip Contents"` is **rejected** on an aspirate.
- `autoSelectPrototype: true` lets the engine choose a technique from liquid type, labware,
  tip type and volume, which is why `prototype` is empty. Set `autoSelectPrototype: false`
  and `prototype` must name a technique that exists in the project.
- `aspirate: true` is what distinguishes this step from Span-8 Dispense — they share one
  implementation.

The matching dispense into the blank plate:

```jsonc
{
  "stepType": "Span-8 Dispense",
  "parameters": {
    "amount": "30",
    "aspirate": false,
    "operation": "Dispense",
    "liquidtype": "Tip Contents",
    "firstWell": 1,
    "useWellExpression": false,
    "spacing": "1",
    "overrideHeight": false,
    "height": 0.0,
    "heightFrom": 1,
    "what": "BCFlat96",
    "where": "BlankCuvettePlate",
    "pod": "Pod1",
    "...": "remaining keys as above"
  }
}
```

`spacing: "1"` on a 96-well plate: consecutive probes step one well apart. Combined with
`firstWell: 1` and `useWellExpression: false`, probes 1–8 land on A1…H1 — column 1. Well
numbering is 1-based and **row-major**, so A1 = 1, A12 = 12, B1 = 13; successive probes step
down a column by +12 wells each. `liquidtype: "Tip Contents"` dispenses what was picked up.

### Placeholder for a device you have no step for

**To actually operate a device, use its add-on step.** A reader, thermocycler, shaker, magnet,
Peltier, and so on are each driven by that device's own **add-on step**, authored like any other
step — consult the device's documentation for its `stepType` and parameters (see
[add-on-steps/](add-on-steps/index.md)). Do not use the pattern below to control a device.

The Comment + Pause pattern here performs **no** device action. It is only a placeholder for a
device you have no authorable step for: it records the intended operation and reserves the plate's
position while an operator performs the read by hand. The absorbance reader in this example has no
step available, so the method marks the read with a [Comment](step-documentation/50-Comment.md) that
names the operation, followed by a [Pause](step-documentation/62-Pause.md) that reserves the plate's
position:

```json
{
  "stepType": "Pause",
  "parameters": {
    "mode": "TimedResource",
    "message": "Paused",
    "time": "60",
    "location": "P8"
  }
}
```

`mode: "TimedResource"` pauses **one resource** — here deck position P8 — for `time`
seconds, leaving the rest of the system free to work in parallel. `message` is not read in
this mode, and `location` is not read in `"PromptedGlobal"` mode, but **all four keys are
emitted anyway**: a JSON import replaces the step's parameter dictionary wholesale, so a key
you leave out is simply absent, and omitting one the chosen mode does read fails at run time.
`mode` is compared literally and is never expression-evaluated — `"=someVar"` always fails.

Then the tips come off:

```json
{
  "stepType": "Span-8 Unload Tips",
  "parameters": {
    "pod": "Pod1",
    "useProbes": { "_biomekType": "comArray", "arraySubtype": "boolean",
                   "values": [true, true, true, true, true, true, true, true] },
    "useExpression": false
  }
}
```

There is no destination key, and that is not an omission — a Span-8 pod has no
return-to-box path for disposable tips. The pod resolves the destination itself from the
box's routing keys or, failing that, the nearest reachable trash.

## 6. The sample sweep: a Loop over plate columns

Twelve passes, four steps each, fresh tips every pass:

```jsonc
{
  "stepType": "Loop",
  "parameters": {
    "variable": "Col",
    "start": "1",
    "end": "12",
    "increment": "1"
  },
  "subSteps": [
    { "stepType": "Span-8 Load Tips", "parameters": { "...": "..." } },
    { "stepType": "Span-8 Aspirate",  "parameters": { "...": "..." } },
    { "stepType": "Span-8 Dispense",  "parameters": { "...": "..." } },
    { "stepType": "Span-8 Unload Tips", "parameters": { "...": "..." } },
    { "stepType": "End", "parameters": {} }
  ]
}
```

- `start`, `end` and `increment` are **strings**, and all three are required. They are read
  with no fallback, so an omitted key imports fine and then fails at run time; the editor's
  own default of `"1"` does not survive a JSON import.
- The last child **must** be `{"stepType": "End", "parameters": {}}`. Loop is a free-form
  container and this is its terminator. Do not confuse the `end` *parameter* (the counter
  bound) with the `"End"` *step* (the structural terminator) — they are unrelated.
- Load Tips and Unload Tips sit **inside** the loop. Move them outside and one set of tips
  visits all 96 sample wells, which is exactly the carryover this assay cannot tolerate.

The loop variable reaches the pipetting steps through the well expression:

```jsonc
{
  "stepType": "Span-8 Aspirate",
  "parameters": {
    "amount": "25",
    "useWellExpression": true,
    "firstWellExpression": "=Col",
    "firstWell": 1,
    "spacing": "1",
    "what": "BCFlat96",
    "where": "SamplePlate",
    "liquidtype": "Well Contents",
    "overrideHeight": false,
    "...": "remaining keys as in the blank aspirate"
  }
}
```

`useWellExpression: true` makes `firstWellExpression` govern and `firstWell` inert. The
expression is the loop variable **on its own** — `"=Col"`. Because well numbering is
row-major, the top well of column *c* is simply well *c*, so `Col` is already the correct
first well. The tempting `"=1+(Col-1)*8"` assumes column-major numbering: it lands the first
probe mid-plate, addresses the wrong wells, and fails outright as soon as fewer than eight
rows remain below the starting well.

Note also that `useWellExpression` (which well) and `useExpression` (which probes) are
independent flags with confusingly similar names. This method sets the first `true` and the
second `false` throughout.

## 7. Normalization: two legs per column

The second loop runs the same 1…12 sweep but does two transfers per pass, with a tip change
between them:

1. Load tips → aspirate 30 µL of sample from `SamplePlate` column `Col` → dispense into
   `DestPlate` column `Col` → unload.
2. Load tips → aspirate 30 µL of diluent from `DiluentReservoir` → dispense into `DestPlate`
   column `Col` → unload.

The tip change between the legs is the point: reusing the sample tips to draw diluent would
put sample into a shared reservoir that every remaining column still has to draw from.

The two aspirates are addressed differently, and the difference is instructive. The sample
aspirate sweeps columns (`useWellExpression: true`, `firstWellExpression: "=Col"`,
`spacing: "1"`). The diluent aspirate does **not** — `useWellExpression: false`,
`firstWell: 1`, `spacing: ""`, plus the same bottom-referenced 2 mm height as the blank
reservoir — because a one-section reservoir has no columns to sweep. Its matching dispense
*does* use `"=Col"`, because the destination is a plate. Source and destination addressing
are independent; do not copy the well expression across a step boundary just because the two
steps are adjacent.

## 8. Finish

```json
{
  "stepType": "Finish",
  "parameters": {
    "bitmap": "OStepUI.ocx,FINISH"
  }
}
```

The minimum. [Finish](step-documentation/02-Finish.md) also accepts `clearDeck`,
`clearDevices` and `cleanupPods` if you want the run to tear down the deck state.

## 9. The volume ledger

Once labware is placed `Known` or `Nominal`, the engine keeps a running per-well volume
across every pipetting step in the method. Two things then fail at enqueue rather than on the
bench: a dispense that would push a well past its class capacity, and an aspirate that would
draw a well below zero. Both errors name the well and the numbers, which makes them easy to
fix but only if the seed you declared in Instrument Setup actually matches what the method
moves.

| Labware | Seed | Draws | Adds | Final | Capacity |
|---------|------|-------|------|-------|----------|
| `SamplePlate`, each well | 200 | −25 (read) −30 (normalize) | — | 145 | 362.76 |
| `CuvettePlate`, each well | 0 | — | +25 | 25 | 362.76 |
| `BlankCuvettePlate`, column 1 | 0 | — | +30 | 30 | 362.76 |
| `DestPlate`, each well | 0 | — | +30 sample, +30 diluent | 60 | 362.76 |
| `BlankReservoir` section | 50 000 | −240 | — | 49 760 | 200 000 |
| `DiluentReservoir` section | 50 000 | −2 880 | — | 47 120 | 200 000 |

Every volume this method moves is ≥ 25 µL, which keeps it inside the working range of the
1000 µL Span-8 syringe techniques and lets `autoSelectPrototype: true` find a match. Drop a
volume below a technique's minimum and auto-selection has nothing to pick.

Worth checking alongside the arithmetic: **what each well ends up holding**, not just how
much. `CuvettePlate` receives only from `SamplePlate` and `BlankCuvettePlate` only from
`BlankReservoir` — one liquid each, because both are read optically. `DestPlate` receives
from two sources, which is the intended dilution. A volume ledger that balances can still
describe a plate that is chemically wrong.

## 10. Complete method JSON

Copy-ready. This is the whole method; the excerpts above are quotations from it.

```json
{
  "format": "Biomek Method",
  "formatVersion": "1.0",
  "author": "",
  "description": "Absorbance concentration check with normalization on i5 Span-8 (disposable tips)",
  "steps": [
    {
      "stepType": "Start",
      "parameters": {
        "bitmap": "OStepUI.ocx,START",
        "let": {},
        "weak": {},
        "prompt": {}
      }
    },
    {
      "stepType": "Instrument Setup",
      "parameters": {
        "barcodeInput?": false,
        "deckItems": {
          "p1": [
            {
              "_biomekType": "labware",
              "class": "LabwareClasses\\BC230_LLS",
              "volumeType": "Unknown",
              "properties": {},
              "tipType": "TipClasses\\T230_LLS"
            }
          ],
          "p2": [
            {
              "_biomekType": "labware",
              "class": "LabwareClasses\\BC230_LLS",
              "volumeType": "Unknown",
              "properties": {},
              "tipType": "TipClasses\\T230_LLS"
            }
          ],
          "p3": [
            {
              "_biomekType": "labware",
              "class": "LabwareClasses\\BCFullReservoir",
              "volumeType": "Known",
              "properties": {
                "name": "BlankReservoir",
                "liquidtype": "Water"
              },
              "evalAmounts": {
                "_biomekType": "comArray",
                "arraySubtype": "variant",
                "values": [50000.0]
              },
              "evalLiquids": {
                "_biomekType": "comArray",
                "arraySubtype": "variant",
                "values": ["Water"]
              }
            }
          ],
          "p4": [
            {
              "_biomekType": "labware",
              "class": "LabwareClasses\\BCFullReservoir",
              "volumeType": "Known",
              "properties": {
                "name": "DiluentReservoir",
                "liquidtype": "Water"
              },
              "evalAmounts": {
                "_biomekType": "comArray",
                "arraySubtype": "variant",
                "values": [50000.0]
              },
              "evalLiquids": {
                "_biomekType": "comArray",
                "arraySubtype": "variant",
                "values": ["Water"]
              }
            }
          ],
          "p5": [],
          "p6": [
            {
              "_biomekType": "labware",
              "class": "LabwareClasses\\BCFlat96",
              "volumeType": "Known",
              "properties": {
                "name": "SamplePlate"
              },
              "evalAmounts": {
                "_biomekType": "comArray",
                "arraySubtype": "variant",
                "values": [
                  200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0,
                  200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0,
                  200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0,
                  200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0,
                  200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0,
                  200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0,
                  200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0,
                  200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0, 200.0
                ]
              },
              "evalLiquids": {
                "_biomekType": "comArray",
                "arraySubtype": "variant",
                "values": [
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water"
                ]
              }
            }
          ],
          "p7": [
            {
              "_biomekType": "labware",
              "class": "LabwareClasses\\BCFlat96",
              "volumeType": "Known",
              "properties": {
                "name": "CuvettePlate"
              },
              "evalAmounts": {
                "_biomekType": "comArray",
                "arraySubtype": "variant",
                "values": [
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0
                ]
              },
              "evalLiquids": {
                "_biomekType": "comArray",
                "arraySubtype": "variant",
                "values": [
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water"
                ]
              }
            }
          ],
          "p8": [
            {
              "_biomekType": "labware",
              "class": "LabwareClasses\\BCFlat96",
              "volumeType": "Known",
              "properties": {
                "name": "BlankCuvettePlate"
              },
              "evalAmounts": {
                "_biomekType": "comArray",
                "arraySubtype": "variant",
                "values": [
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0
                ]
              },
              "evalLiquids": {
                "_biomekType": "comArray",
                "arraySubtype": "variant",
                "values": [
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water"
                ]
              }
            }
          ],
          "p9": [],
          "p10": [],
          "p11": [
            {
              "_biomekType": "labware",
              "class": "LabwareClasses\\BCFlat96",
              "volumeType": "Known",
              "properties": {
                "name": "DestPlate"
              },
              "evalAmounts": {
                "_biomekType": "comArray",
                "arraySubtype": "variant",
                "values": [
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
                  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0
                ]
              },
              "evalLiquids": {
                "_biomekType": "comArray",
                "arraySubtype": "variant",
                "values": [
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water",
                  "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water", "Water"
                ]
              }
            }
          ],
          "p12": [],
          "p13": [],
          "p14": [],
          "p15": [],
          "p16": [
            {
              "_biomekType": "labware",
              "class": "LabwareClasses\\BC230_LLS",
              "volumeType": "Unknown",
              "properties": {},
              "tipType": "TipClasses\\T230_LLS"
            }
          ],
          "p17": [
            {
              "_biomekType": "labware",
              "class": "LabwareClasses\\BC230_LLS",
              "volumeType": "Unknown",
              "properties": {},
              "tipType": "TipClasses\\T230_LLS"
            }
          ],
          "p18": [],
          "p19": [],
          "p20": []
        },
        "layout": "Span8",
        "pause?": true,
        "podSetup": {
          "leftHasTips": false,
          "leftTipType": "",
          "rightHasTips": false,
          "rightTipType": ""
        },
        "verifyPodSetup?": true
      }
    },
    {
      "stepType": "Comment",
      "parameters": {
        "description": "Reader cartridges are represented by two 96-well plates",
        "comment": "The reader's cuvette cartridge is not a catalog labware class, so BCFlat96 stands in for it. Two plates are placed: BlankCuvettePlate at P8 holds the blank, CuvettePlate at P7 holds all 96 samples. One 96-well plate has no spare well for the blank once 96 samples occupy it, and a well that is read optically must contain exactly one liquid. BCFlat96 holds 362.76 uL per well, well above the 30 uL cuvette load, so the well geometry and the volume model stay valid."
      }
    },
    {
      "stepType": "Comment",
      "parameters": {
        "description": "Review measurement parameters before the run",
        "comment": "Operator checkpoint: confirm wavelengths, target concentration, and the active sample set before any liquid moves."
      }
    },
    {
      "stepType": "Span-8 Load Tips",
      "parameters": {
        "pod": "Pod1",
        "tips": "BC230_LLS",
        "useProbes": {
          "_biomekType": "comArray",
          "arraySubtype": "boolean",
          "values": [true, true, true, true, true, true, true, true]
        },
        "useExpression": false
      }
    },
    {
      "stepType": "Span-8 Aspirate",
      "parameters": {
        "amount": "30",
        "aspirate": true,
        "autoSelectPrototype": true,
        "customHeight": false,
        "discardExcess": false,
        "emptyTips": false,
        "firstWell": 1,
        "firstWellExpression": "",
        "height": 2.0,
        "heightFrom": 1,
        "liquidtype": "Well Contents",
        "mandrelExpression": "",
        "operation": "Aspirate",
        "overrideHeight": true,
        "pod": "Pod1",
        "prototype": "",
        "refreshTips": false,
        "spacing": "",
        "tipLabwareClass": "",
        "useExpression": false,
        "useProbes": {
          "_biomekType": "comArray",
          "arraySubtype": "boolean",
          "values": [true, true, true, true, true, true, true, true]
        },
        "useWellExpression": false,
        "what": "BCFullReservoir",
        "where": "BlankReservoir"
      }
    },
    {
      "stepType": "Span-8 Dispense",
      "parameters": {
        "amount": "30",
        "aspirate": false,
        "autoSelectPrototype": true,
        "customHeight": false,
        "discardExcess": false,
        "emptyTips": false,
        "firstWell": 1,
        "firstWellExpression": "",
        "height": 0.0,
        "heightFrom": 1,
        "liquidtype": "Tip Contents",
        "mandrelExpression": "",
        "operation": "Dispense",
        "overrideHeight": false,
        "pod": "Pod1",
        "prototype": "",
        "spacing": "1",
        "useExpression": false,
        "useProbes": {
          "_biomekType": "comArray",
          "arraySubtype": "boolean",
          "values": [true, true, true, true, true, true, true, true]
        },
        "useWellExpression": false,
        "what": "BCFlat96",
        "where": "BlankCuvettePlate"
      }
    },
    {
      "stepType": "Comment",
      "parameters": {
        "description": "Blank read",
        "comment": "The reader measures BlankCuvettePlate at P8 at 260, 280 and 230 nm and records the baseline optical density that is subtracted from every sample reading. The reader is driven outside the method; the Pause below reserves P8 for the 60 s read."
      }
    },
    {
      "stepType": "Pause",
      "parameters": {
        "mode": "TimedResource",
        "message": "Paused",
        "time": "60",
        "location": "P8"
      }
    },
    {
      "stepType": "Span-8 Unload Tips",
      "parameters": {
        "pod": "Pod1",
        "useProbes": {
          "_biomekType": "comArray",
          "arraySubtype": "boolean",
          "values": [true, true, true, true, true, true, true, true]
        },
        "useExpression": false
      }
    },
    {
      "stepType": "Comment",
      "parameters": {
        "description": "Sample loading: one column per pass",
        "comment": "Twelve passes sweep the sample plate column by column. Fresh tips on every pass eliminate carryover between columns."
      }
    },
    {
      "stepType": "Loop",
      "parameters": {
        "variable": "Col",
        "start": "1",
        "end": "12",
        "increment": "1"
      },
      "subSteps": [
        {
          "stepType": "Span-8 Load Tips",
          "parameters": {
            "pod": "Pod1",
            "tips": "BC230_LLS",
            "useProbes": {
              "_biomekType": "comArray",
              "arraySubtype": "boolean",
              "values": [true, true, true, true, true, true, true, true]
            },
            "useExpression": false
          }
        },
        {
          "stepType": "Span-8 Aspirate",
          "parameters": {
            "amount": "25",
            "aspirate": true,
            "autoSelectPrototype": true,
            "customHeight": false,
            "discardExcess": false,
            "emptyTips": false,
            "firstWell": 1,
            "firstWellExpression": "=Col",
            "height": 0.0,
            "heightFrom": 0,
            "liquidtype": "Well Contents",
            "mandrelExpression": "",
            "operation": "Aspirate",
            "overrideHeight": false,
            "pod": "Pod1",
            "prototype": "",
            "refreshTips": false,
            "spacing": "1",
            "tipLabwareClass": "",
            "useExpression": false,
            "useProbes": {
              "_biomekType": "comArray",
              "arraySubtype": "boolean",
              "values": [true, true, true, true, true, true, true, true]
            },
            "useWellExpression": true,
            "what": "BCFlat96",
            "where": "SamplePlate"
          }
        },
        {
          "stepType": "Span-8 Dispense",
          "parameters": {
            "amount": "25",
            "aspirate": false,
            "autoSelectPrototype": true,
            "customHeight": false,
            "discardExcess": false,
            "emptyTips": false,
            "firstWell": 1,
            "firstWellExpression": "=Col",
            "height": 0.0,
            "heightFrom": 1,
            "liquidtype": "Tip Contents",
            "mandrelExpression": "",
            "operation": "Dispense",
            "overrideHeight": false,
            "pod": "Pod1",
            "prototype": "",
            "spacing": "1",
            "useExpression": false,
            "useProbes": {
              "_biomekType": "comArray",
              "arraySubtype": "boolean",
              "values": [true, true, true, true, true, true, true, true]
            },
            "useWellExpression": true,
            "what": "BCFlat96",
            "where": "CuvettePlate"
          }
        },
        {
          "stepType": "Span-8 Unload Tips",
          "parameters": {
            "pod": "Pod1",
            "useProbes": {
              "_biomekType": "comArray",
              "arraySubtype": "boolean",
              "values": [true, true, true, true, true, true, true, true]
            },
            "useExpression": false
          }
        },
        {
          "stepType": "End",
          "parameters": {}
        }
      ]
    },
    {
      "stepType": "Comment",
      "parameters": {
        "description": "Sample read",
        "comment": "The reader measures CuvettePlate at P7 at 260, 280 and 230 nm and records per-well concentration and the A260/A280 and A260/A230 purity ratios. The Pause below reserves P7 for the 120 s read."
      }
    },
    {
      "stepType": "Pause",
      "parameters": {
        "mode": "TimedResource",
        "message": "Paused",
        "time": "120",
        "location": "P7"
      }
    },
    {
      "stepType": "Comment",
      "parameters": {
        "description": "Normalization volumes",
        "comment": "Per-well sample and diluent volumes are computed from the recorded concentrations and the operator's target concentration. The transfers below move a fixed 30 uL of sample plus 30 uL of diluent; to use computed volumes, drive each step's amount from an expression instead."
      }
    },
    {
      "stepType": "Loop",
      "parameters": {
        "variable": "Col",
        "start": "1",
        "end": "12",
        "increment": "1"
      },
      "subSteps": [
        {
          "stepType": "Span-8 Load Tips",
          "parameters": {
            "pod": "Pod1",
            "tips": "BC230_LLS",
            "useProbes": {
              "_biomekType": "comArray",
              "arraySubtype": "boolean",
              "values": [true, true, true, true, true, true, true, true]
            },
            "useExpression": false
          }
        },
        {
          "stepType": "Span-8 Aspirate",
          "parameters": {
            "amount": "30",
            "aspirate": true,
            "autoSelectPrototype": true,
            "customHeight": false,
            "discardExcess": false,
            "emptyTips": false,
            "firstWell": 1,
            "firstWellExpression": "=Col",
            "height": 0.0,
            "heightFrom": 0,
            "liquidtype": "Well Contents",
            "mandrelExpression": "",
            "operation": "Aspirate",
            "overrideHeight": false,
            "pod": "Pod1",
            "prototype": "",
            "refreshTips": false,
            "spacing": "1",
            "tipLabwareClass": "",
            "useExpression": false,
            "useProbes": {
              "_biomekType": "comArray",
              "arraySubtype": "boolean",
              "values": [true, true, true, true, true, true, true, true]
            },
            "useWellExpression": true,
            "what": "BCFlat96",
            "where": "SamplePlate"
          }
        },
        {
          "stepType": "Span-8 Dispense",
          "parameters": {
            "amount": "30",
            "aspirate": false,
            "autoSelectPrototype": true,
            "customHeight": false,
            "discardExcess": false,
            "emptyTips": false,
            "firstWell": 1,
            "firstWellExpression": "=Col",
            "height": 0.0,
            "heightFrom": 1,
            "liquidtype": "Tip Contents",
            "mandrelExpression": "",
            "operation": "Dispense",
            "overrideHeight": false,
            "pod": "Pod1",
            "prototype": "",
            "spacing": "1",
            "useExpression": false,
            "useProbes": {
              "_biomekType": "comArray",
              "arraySubtype": "boolean",
              "values": [true, true, true, true, true, true, true, true]
            },
            "useWellExpression": true,
            "what": "BCFlat96",
            "where": "DestPlate"
          }
        },
        {
          "stepType": "Span-8 Unload Tips",
          "parameters": {
            "pod": "Pod1",
            "useProbes": {
              "_biomekType": "comArray",
              "arraySubtype": "boolean",
              "values": [true, true, true, true, true, true, true, true]
            },
            "useExpression": false
          }
        },
        {
          "stepType": "Span-8 Load Tips",
          "parameters": {
            "pod": "Pod1",
            "tips": "BC230_LLS",
            "useProbes": {
              "_biomekType": "comArray",
              "arraySubtype": "boolean",
              "values": [true, true, true, true, true, true, true, true]
            },
            "useExpression": false
          }
        },
        {
          "stepType": "Span-8 Aspirate",
          "parameters": {
            "amount": "30",
            "aspirate": true,
            "autoSelectPrototype": true,
            "customHeight": false,
            "discardExcess": false,
            "emptyTips": false,
            "firstWell": 1,
            "firstWellExpression": "",
            "height": 2.0,
            "heightFrom": 1,
            "liquidtype": "Well Contents",
            "mandrelExpression": "",
            "operation": "Aspirate",
            "overrideHeight": true,
            "pod": "Pod1",
            "prototype": "",
            "refreshTips": false,
            "spacing": "",
            "tipLabwareClass": "",
            "useExpression": false,
            "useProbes": {
              "_biomekType": "comArray",
              "arraySubtype": "boolean",
              "values": [true, true, true, true, true, true, true, true]
            },
            "useWellExpression": false,
            "what": "BCFullReservoir",
            "where": "DiluentReservoir"
          }
        },
        {
          "stepType": "Span-8 Dispense",
          "parameters": {
            "amount": "30",
            "aspirate": false,
            "autoSelectPrototype": true,
            "customHeight": false,
            "discardExcess": false,
            "emptyTips": false,
            "firstWell": 1,
            "firstWellExpression": "=Col",
            "height": 0.0,
            "heightFrom": 1,
            "liquidtype": "Tip Contents",
            "mandrelExpression": "",
            "operation": "Dispense",
            "overrideHeight": false,
            "pod": "Pod1",
            "prototype": "",
            "spacing": "1",
            "useExpression": false,
            "useProbes": {
              "_biomekType": "comArray",
              "arraySubtype": "boolean",
              "values": [true, true, true, true, true, true, true, true]
            },
            "useWellExpression": true,
            "what": "BCFlat96",
            "where": "DestPlate"
          }
        },
        {
          "stepType": "Span-8 Unload Tips",
          "parameters": {
            "pod": "Pod1",
            "useProbes": {
              "_biomekType": "comArray",
              "arraySubtype": "boolean",
              "values": [true, true, true, true, true, true, true, true]
            },
            "useExpression": false
          }
        },
        {
          "stepType": "End",
          "parameters": {}
        }
      ]
    },
    {
      "stepType": "Comment",
      "parameters": {
        "description": "Final data review",
        "comment": "Normalized volumes, concentrations and purity ratios are reviewed and archived before the method ends."
      }
    },
    {
      "stepType": "Finish",
      "parameters": {
        "bitmap": "OStepUI.ocx,FINISH"
      }
    }
  ]
}
```

## 11. Adapting it

- **Fewer than 12 columns.** Change the two Loops' `end`. If the count is a run-time input,
  declare it in Start's `let` and write `"end": "=NumCols"`.
- **Per-well normalization volumes.** The transfers here move a fixed 30 µL of sample and
  30 µL of diluent. `amount` is expression-capable on both Span-8 Aspirate and Span-8
  Dispense, so a computed volume goes in as `"amount": "=SampleVolume"` once the value is in
  scope.
- **Collapse a leg into one step.** Load Tips + Aspirate + Dispense + Unload Tips is a
  [Transfer](step-documentation/11-Transfer.md) written out longhand. A single Transfer step
  handles tip loading, the aspirate/dispense pair, and tip disposal, and takes source and
  destination as entries in its `items` array (`position` / `labwareClass`, not
  `where` / `what`). The longhand form is used here because the two loops need a tip change
  at a specific point inside the pass and because the blank leg has a device read wedged
  between its dispense and its unload.
- **Move to a Multichannel pod.** Swap every `Span-8 *` step type for its `Multichannel *`
  equivalent and rename the position and class keys: Multichannel steps use `location` and
  `labwareClass` where Span-8 uses `where` and `what`. Reservoir aspiration and
  bottom-referenced heights carry over; per-probe selection does not, and neither does
  liquid-level sensing — Multichannel and Fixed-8 pods cannot sense a surface, so on those
  pods a well pipetted with a **liquid-relative** technique (one that references off the liquid
  surface) must be declared at least `Nominal`. A from-bottom or fixed-height technique
  references off the well itself and has no such requirement.

## 12. Further reading

Background for the choices made above:

- [Labware & Position Resolution](concept-guides/02-labware-and-position-resolution.md) —
  position vs. class, the per-family key spellings, name resolution, reachability.
- [Well Volume Tracking](concept-guides/05-well-volume-tracking.md) — the running per-well
  ledger the table in §9 replays.
- [Expressions](concept-guides/01-expressions.md) and
  [The Variable Environment](concept-guides/04-variable-environment.md) — `=Col`, `let`, and
  what is in scope where.
- [Pipetting Techniques and Templates](Pipetting-Techniques-and-Templates.md) —
  `autoSelectPrototype`, technique naming, height override, liquid types and volume types.
- [Catalog: Labware Classes](biomek-file-formats/project-items/catalog-labware-classes.md)
  and [Catalog: Tip Classes](biomek-file-formats/project-items/catalog-tip-classes.md) —
  well counts, capacities, and the box-to-tip-class pairing used in §2.
