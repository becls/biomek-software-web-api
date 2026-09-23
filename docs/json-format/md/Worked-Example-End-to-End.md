# Worked Example: Simple i3 Fixed-8 Method

A complete, copy-ready Biomek method JSON that transfers liquid from one plate to another on an i3 instrument with a Fixed-8 pod. Every position name, labware class, and tip type used here is valid for the i3 default deck and project.

## What each step does

1. **Start** -- initializes the method. The `let`, `weak`, and `prompt` dictionaries are empty (no variables), and the required `bitmap` icon reference is included.
2. **Instrument Setup** -- selects the `"i3"` deck layout and places three items: a BC230 tip box at P7, a BCFlat96 source plate at P3 pre-filled with 200 uL of Water in every well (`volumeType: "Known"`, `evalAmounts`/`evalLiquids`), and an empty BCFlat96 destination plate at P9 with defined-but-zero volumes (`volumeType: "Known"`, `evalAmounts` all 0.0, `evalLiquids` all `"Water"`). The destination is `"Known"` (not `"Unknown"`) because the default `"F8 Medium"` technique moves relative to the liquid level, which Fixed-8 cannot sense at run time -- see [Pipetting Techniques and Templates](Pipetting-Techniques-and-Templates.md#liquid-types) "Liquid types". All other positions are declared empty, and `podSetup` confirms no tips are pre-loaded.
3. **Fixed-8 Load Tips** -- picks up 8 disposable tips from the tip box at P7.
4. **Fixed-8 Aspirate** -- aspirates 100 uL from column 1 of the source plate at P3 (wells A1 through H1, i.e. `firstWell: 1`). Uses the `"F8 Medium"` technique.
5. **Fixed-8 Dispense** -- dispenses 100 uL into column 1 of the destination plate at P9. Uses the same technique.
6. **Fixed-8 Unload Tips** -- returns the tips to the box they came from.
7. **Finish** -- parks the pod, clears the deck, and ends the method.

## Complete method JSON

```json
{
  "format": "Biomek Method",
  "formatVersion": "1.0",
  "author": "",
  "description": "Simple column-1 plate-to-plate transfer on i3 Fixed-8",
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
          "p1": [],
          "p2": [],
          "p3": [
            {
              "_biomekType": "labware",
              "class": "LabwareClasses\\BCFlat96",
              "volumeType": "Known",
              "properties": {
                "name": "SourcePlate"
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
          "p4": [],
          "p5": [],
          "p6": [],
          "p7": [
            {
              "_biomekType": "labware",
              "class": "LabwareClasses\\BC230",
              "volumeType": "Unknown",
              "properties": {},
              "tipType": "TipClasses\\T230"
            }
          ],
          "p8": [],
          "p9": [
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
          "p10": [],
          "p11": [],
          "p12": [],
          "s1": [],
          "s2": [],
          "s3": []
        },
        "layout": "i3",
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
      "stepType": "Fixed-8 Load Tips",
      "parameters": {
        "tips": "P7",
        "mode": "NumberOfTips",
        "numberOfTips": "8"
      }
    },
    {
      "stepType": "Fixed-8 Aspirate",
      "parameters": {
        "pod": "Pod1",
        "where": "SourcePlate",
        "what": "BCFlat96",
        "operation": "Aspirate",
        "amount": "100",
        "firstWell": 1,
        "useWellExpression": false,
        "liquidType": "Well Contents",
        "autoSelectPrototype": false,
        "prototype": "F8 Medium"
      }
    },
    {
      "stepType": "Fixed-8 Dispense",
      "parameters": {
        "pod": "Pod1",
        "where": "DestPlate",
        "what": "BCFlat96",
        "operation": "Dispense",
        "amount": "100",
        "firstWell": 1,
        "useWellExpression": false,
        "liquidType": "Tip Contents",
        "autoSelectPrototype": false,
        "prototype": "F8 Medium"
      }
    },
    {
      "stepType": "Fixed-8 Unload Tips",
      "parameters": {
        "tipDestination": "<where they came from>"
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

## Common adjustments

- **Swap to a different instrument.** Replace `"layout": "i3"` with the target deck (e.g. `"Span8"` for an i5 Span-8, `"Multichannel"` for an i5 MC), swap all Fixed-8 step types for the matching family (`Span-8 Load Tips` / `Span-8 Aspirate` / `Span-8 Dispense` / `Span-8 Unload Tips`, or `Multichannel Load Tips` / `Multichannel Aspirate` / `Multichannel Dispense` / `Multichannel Unload Tips`), and replace the technique name with one built for the target pod family (`"S8 1000 Medium"` for Span-8, `"MC P60"` for Multichannel). Move the tip box to a valid tip-load position for the target deck (e.g. a `TL*` position on the MC deck).
- **Change volumes.** Edit `"amount"` in both the Aspirate and Dispense steps. If the source plate's tracked volumes change, update the `evalAmounts` array to match.
- **Transfer multiple columns.** Wrap the Aspirate and Dispense steps inside a [Loop](step-documentation/58-Loop.md) with `"variable": "Col"`, `"start": "1"`, `"end": "12"`, `"increment": "1"` and set `useWellExpression: true` with `firstWellExpression: "=Col"` on each pipetting step so the pod sweeps columns 1 through 12. To get fresh tips per column, move the Fixed-8 Load Tips and Fixed-8 Unload Tips steps *inside* the loop.
- **Use a Transfer step instead.** A single [Transfer](step-documentation/11-Transfer.md) step can replace the Load Tips + Aspirate + Dispense + Unload Tips sequence. Transfer handles tip loading, aspiration, dispensing, and tip disposal in one step -- see the Transfer step reference for the full parameter set.

## Related

Each step above is documented under [step-documentation/](step-documentation/). For the Instrument Setup labware/position model, see [Labware and Position Resolution](concept-guides/02-labware-and-position-resolution.md); for technique naming conventions (`F8 Medium`, etc.), see [Pipetting Techniques and Templates](Pipetting-Techniques-and-Templates.md).
