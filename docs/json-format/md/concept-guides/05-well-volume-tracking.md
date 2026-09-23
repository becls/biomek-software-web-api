# Concept Guide: Well Volume Tracking

Biomek keeps a running per-well volume for every labware whose contents it knows
about, and it maintains that ledger **while the method is being enqueued** —
before anything moves. Steps debit it when they aspirate and credit it when they
dispense: a dispense that would push a well above the labware's capacity fails
at enqueue, and an aspirate that would drive a tracked well below empty fails at
enqueue.

This is the mechanism behind two of the most common enqueue errors:

```
Cannot pipette 50.000 uL; the well only has 0.000 uL in it.
Well volume out of allowed range: 3000.000 uL.
```

Neither is a run-time warning and neither can be suppressed. Both are reported against a position,
not against the step that caused the imbalance, so the step named in the error is often the first
one to notice a shortfall created several steps earlier.

## 1. What seeds the ledger

The ledger starts at whatever [Instrument Setup](../step-documentation/03-Instrument-Setup.md)
declares. Two keys on the labware object do it:

- `volumeType` — `"Known"` or `"Nominal"` puts the labware under tracking. `"Unknown"` leaves it
  untracked.
- `evalAmounts` — the per-well starting volumes, row-major, in µL.

A tracked labware with no `evalAmounts` starts every well at **0**. That is the single most common
cause of the "well only has 0.000 uL" error: the plate was filled by an operator off-deck, the
method never says so, and the first aspirate is drawing from a well the engine believes is empty.
**If liquid arrives on the deck by any route other than a pipetting step in this method, you must
seed it here** — an off-deck pre-fill, a plate carried over from a previous method, or a reagent the
protocol assumes is already in the reservoir.

The array length must equal the labware's well count exactly — there is no shorthand or replication.
See §`evalAmounts` / `evalLiquids` in
[Instrument Setup](../step-documentation/03-Instrument-Setup.md)
for the per-array error messages and the row-major indexing convention.

## 2. What moves it

Every pipetting step that moves liquid updates the ledger: Transfer, Combine, the Fixed-8, Span-8
and Multichannel Aspirate/Dispense/Mix steps, Serial Dilution, and Select Tips Aspirate/Dispense/Mix
steps nested inside a Select Tips container.
Aspirates debit the source; dispenses credit the destination.

Two consequences catch people out:

- **The engine debits what the technique actually draws, not the volume you wrote.** The ledger sees
  the larger number, so a source well sized exactly to the sum of the item volumes can still run
  short. Two technique settings add to the draw, and `splitVolume` transfers carry the overhead once
  per split cycle — see "Sizing a source well" below.
- **Destinations become tracked whether or not you declared them.** Once liquid has been dispensed
  into a plate, its wells carry tracked volumes from that point on, even if it was placed
  `"Unknown"`. A later aspirate from that plate is then subject to the same below-empty check.

### Sizing a source well against the real draw

Exactly two technique settings inflate the source-well debit above the volume you asked for:

| Setting (technique **General** tab) | Effect on the draw |
|---|---|
| **Pre-wet** + **Pre-wet overage** | When pre-wet is on, the pre-wet cycle draws `volume + overage` from the source. It is returned to the same well immediately afterward, but the check runs on the draw, so the well must hold the larger amount at that moment. |
| **Conditioning excess**, **Conditioning section count**, **Conditioning section volume** | Every aspirate draws `volume + (count × section volume) + excess`. The excess may be an absolute µL figure or a percentage of the transfer volume (e.g. `15%`), so it does not always scale the same way across techniques. |

The projected draw for one aspirate is the **larger** of those two expressions
or the specified volume. If the technique's **Override liquid type
settings** is off, the liquid type supplies the pre-wet overage instead; no other liquid-type
setting reaches this calculation.

**Air gaps are not part of it.** A leading or trailing air gap and the blowout are drawn from air,
not from the well, and never debit the ledger. Sizing a source well for them wastes reagent.

Worked example — a 25 µL aspirate under a technique with pre-wet on, a pre-wet overage of `1.0`, and
no conditioning, from a well seeded at exactly `25.0`:

```text
projected draw = 25 + 1 = 26 µL
Cannot pipette 26.000 µL; the well only has 25.000 µL in it.
```

Seeding that well at `26.0` or higher clears it.

One caveat this arithmetic does not cover: a technique's **calibration slope** and **calibration
offset** scale what the pump physically pulls, but they do not enter the projection above. A
technique with a slope above `1.0` draws more than the ledger believes, so the well can run genuinely
low at run time without any enqueue error. Leave a margin rather than seeding to the exact
projected figure.

### A mix is bounded by what the well holds, not by what it can hold

Each mix cycle is a real aspirate from the well followed by a real dispense back into it, so the
aspirate half is checked like any other:

```
Cannot pipette 200.000 uL; the well only has 150.000 uL in it.
```

The per-cycle mix volume must be no larger than what the well contains **at that point in the
method** — not what the labware class can hold. A 200 µL mix in a 300 µL well is fine only if 200 µL
is actually present when the mix runs.

- **Cycle count is free.** Every cycle returns its draw to the same well, so the net ledger change
  across a mix is zero. Ten cycles of 150 µL need 150 µL present, not 1500 µL.
- **No technique overhead is added.** The pre-wet and conditioning terms from "Sizing a source well"
  above do not inflate a mix cycle.
- **Where the mix sits decides its budget.** A technique that mixes on the *aspirate* side mixes the
  source before the transfer draws from it, so its budget is the source's full contents. A technique
  that mixes on the *dispense* side mixes the destination after the delivery lands, so its budget
  includes the volume just delivered — into a well that started empty, that is the transfer volume.
  A 100 µL transfer cannot be followed by a 150 µL mix at the destination.

Separately, a mix requires the tips to be empty of liquid before it starts:

```
The mix operation cannot begin until tips are empty.
```

Air gaps do not trip this; anything classified as a liquid does. Most often seen when a step reuses
tips that still hold contents — an aspirate followed by a standalone Mix step on the same tips, with
nothing in between to discharge them.

## 3. Where the ceiling comes from

The upper bound is the labware class's per-well maximum — per **section** for a reservoir. It is
checked both when you seed the labware and every time a dispense credits a well. The capacities are
in the per-type tables in
[Catalog: Labware Classes](../biomek-file-formats/project-items/catalog-labware-classes.md).

Reservoirs are addressed by section rather than by well throughout: a one-section reservoir has one
tracked volume, not 96.

## 4. Diagnosing a below-empty failure

The error names a position but not a well, so the first job is to work out *which* well it means.
Resist the urge to raise every source volume in the method — that is the most common wasted attempt,
and it cannot succeed against the case in step 2.

1. **Was the well ever seeded?** If the liquid gets there off-deck, add `volumeType` and
   `evalAmounts` in Instrument Setup. This is the most common answer.
2. **Is it a well you never seeded at all?** Every well that has received a dispense is tracked from
   that point on, so the shortfall is often in an **intermediate** well — a dilution plate, a
   mastermix well, a pooling target — whose entire contents came from an earlier step in this
   method. No amount of over-provisioning your *sources* reaches it; the fix is upstream, in
   whatever dispensed too little. **The reported figure is the clue.** A message naming an odd
   amount, `7.500` rather than something you seeded, points at a computed intermediate: search the
   method for the step that would leave exactly that much behind.
3. **Does the arithmetic close?** Add up every draw from that well across the whole method,
   including technique overhead and every split cycle, and compare against the seeded volume.
4. **Is a residual or floor eating the difference?** If the method leaves a deliberate residual in
   the source, the seeded volume has to cover the draws *plus* the residual.
5. **Is it a reservoir being addressed as a plate?** A 96-element seed on a one-section reservoir is
   a length error, not a volume error, and reports differently.
6. **Did an earlier step drain it?** The ledger is cumulative across the method. A well topped up
   by an earlier dispense is only as full as that dispense left it.

## 5. Quick checklist

- Seed any labware that starts with liquid in it — tracked labware defaults to empty, not to
  unknown.
- Seed destinations too, with `0.0`, when the technique follows the liquid level.
- Size sources against the technique's real draw, not the nominal item volume.
- Size mix volumes against what the well holds when the mix runs, not against the labware's capacity.
- Check reservoir seeds are section-length.
- Treat a below-empty error as an accounting question about the whole method, not about the step
  that reported it.
