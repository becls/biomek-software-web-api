# Pod Reach Envelopes (i7 worked example)

Which pod can reach which position is decided first by each pod's **X-axis travel
envelope** — a hard min/max on where the pipettor can go. This page publishes those
envelopes for the default i7 instruments, shows where to read the equivalent numbers for
**your** instrument, and turns them into a position-by-position reach table for the i7
Hybrid so that "I need a Span-8 tip box on an i7" has a deck position that actually works.

For the mechanism behind these numbers and every other reach failure mode, see
[Reachability and Access](./reachability-and-access.md). For the position coordinates,
see the [samples/](./samples/) deck views.

> **These numbers are instrument-specific.** They are read from the default i7 exports and
> are correct for those files. Your instrument's envelope depends on its chassis, installed
> pods, and calibration — read yours as shown under "Reading your own envelope" rather than
> copying a value here.

## Reading your own envelope

Every pod in a Biomek Instrument Settings export carries its X travel range in two places:

- **Human-readable:** `podSettings.<pod>.asText` contains an *Axis Limit Settings* block,
  e.g. `X: [ 48.59 , 146.34 ]`. This is the same text shown in the Biomek software's Pod
  Settings dialog, so you can read it without exporting.
- **Exact:** `podSettings.<pod>.settings.min.x` and `.settings.max.x` (cm), alongside `.y`
  and the other axes.

## X travel envelopes, default i7 instruments

Read from each instrument's export (`podSettings.<pod>.settings.{min,max}.x`). Values in cm, in
each pod's own pipettor (carriage-axis) frame.

| Instrument (export) | Pod | Head | X min | X max |
|---|---|---|---|---|
| i7 Multichannel (`biomek-i7-multichannel.json`) | Pod1 | MC 96 | 10.52 | 134.665 |
| i7 Span-8 (`biomek-i7-span8.json`) | Pod1 | Span-8 | 10.116 | 146.336 |
| i7 Hybrid (`biomek-i7-hybrid.json`) | Pod1 | MC 96 | 10.52 | 108.269 |
| i7 Hybrid (`biomek-i7-hybrid.json`) | Pod2 | Span-8 | 48.587 | 146.336 |
| i7 DualMultichannel (`biomek-i7-dualmultichannel.json`) | Pod1 | MC 96 | 10.52 | 95.499 |
| i7 DualMultichannel (`biomek-i7-dualmultichannel.json`) | Pod2 | MC 96 | 49.462 | 134.440 |

Y is far less constraining side-to-side but bounds the front-to-back reach: on the i7 Hybrid
Pod1 Y is `[14.60, 59.59]` and Pod2 Y is `[15.11, 66.33]`. The rearmost deck row can exceed a
pod's Y once the head's own depth is added — see the back-row note in
[Reachability and Access](./reachability-and-access.md).

### What "two pods on one bridge" costs, in numbers

Adding a second pod narrows the first one's travel — the same instrument model loses reach on
the shared side:

- **MC pod:** reaches X up to **134.665** cm alone (i7 Multichannel), but only to **95.499**
  cm as the left pod of the DualMultichannel — ~39 cm of right-side travel gone.
- **Span-8 pod:** reaches X down to **10.116** cm alone (i7 Span-8), but only to **48.587** cm
  as the right pod of the Hybrid — ~38 cm of left-side travel gone, which is why the left of a
  Hybrid deck is out of Span-8 reach (below).

These are the static axis limits recorded per instrument; the live two-pod interaction can
narrow reach *further* still, depending on where the other pod is parked. Confirm borderline
positions in Manual Control.

## The travel-range error: reading the two numbers

Assigning a position to a pod that cannot reach it is caught when the method is enqueued, and
the message quotes the axis limit back at you:

```
Unable to find a path for pod2 pipettor to approach position P8:
Specified pipettor destination X 27.270 cm is outside of travel range,
which is between 48.587 cm and 146.336 cm.
```

Read it in four parts:

- **`pod2`** — the pod the step was assigned to. Its envelope is the one that applies; the other
  pod's envelope has no bearing on this message.
- **`P8`** — the position the step named.
- **`destination X 27.270 cm`** — where that position puts the pipettor, expressed in the pod's
  own axis frame. This is *not* the position's `x1`; head and mandrel geometry offset it, by a
  fixed per-pod amount the export does not publish.
- **`between 48.587 cm and 146.336 cm`** — the pod's X envelope, exactly the
  `podSettings.<pod>.settings.min.x` / `.max.x` pair. Match it against the table above to confirm
  which pod you are looking at: on the default i7 Hybrid, `[48.587, 146.336]` is Pod2, the Span-8.

Despite the opening clause, this is **not** an obstruction and not a path-planning problem.
Nothing about the deck's *contents* is being complained about, so emptying neighboring
positions, changing the load order, reordering steps or re-framing will not change the outcome.
The requested coordinate is outside a hard axis limit. Either move the labware to a position
inside that pod's envelope, or assign the step to the pod whose envelope contains the
position.

The same message is emitted per axis, with the axis letter naming which envelope row to check —
X against `min.x`/`max.x`, Y against `min.y`/`max.y`. A multichannel pod reaching for a back-row
position reports the Y form:

```
Unable to find a path for pod1 pipettor to approach position TL5:
Specified pipettor destination Y 64.649 cm is outside of travel range,
which is between 14.596 cm and 59.591 cm.
```

## Each pod has its own envelope

On a single-pod instrument there is one envelope, and "can this instrument reach this position"
has one answer. On a two-pod instrument there are **two envelopes, and they overlap only
partially** — so a position one pod reaches comfortably is routinely outside the other pod's
range. This is the normal case, not an edge case. On the default i7 Hybrid, Pod1 spans
`[10.52, 108.269]` and Pod2 spans `[48.587, 146.336]`: the left of the deck belongs to Pod1
alone, the right to Pod2 alone, and only the middle is shared.

The consequence for authoring is that a deck plan for a two-pod instrument is not one allocation
of labware to positions — it is **one allocation per pod**. Labware only Pod2 touches must sit
inside Pod2's envelope; labware only Pod1 touches must sit inside Pod1's; labware both pods touch
must sit inside the intersection. Positions have to be chosen per pod, and that makes authoring
for a two-pod instrument materially harder than for a single-pod one, on the same deck, with the
same labware.

The position that "looks central" is the one that most often fails. A position whose pipettor
destination lands at 46.800 cm is under 2 cm short of Pod2's 48.587 cm minimum — visually
mid-deck, and still unreachable. Nothing in the deck layout marks these positions out; they are
listed exactly like every other position, with the same `characteristics` keys.

## Before you assign a position to a pod

Do this per pod, for every pod that touches the labware — including the pod that fetches its tip
box, which is easy to forget when the tip source is a different position from the pipetting
target.

1. **Identify the pod.** Take the pod the step names (`pod1` / `pod2`), not "the instrument".
2. **Read that pod's envelope.** `podSettings.<pod>.settings.min.x` and `.max.x` from your
   instrument's export, or the *Axis Limit Settings* block in `podSettings.<pod>.asText`, or the
   Pod Settings dialog in the software. Read the second pod's separately — do not reuse the
   first's.
3. **Read the position's geometry.** `x1` for pod 1, `x2` for pod 2, and `xSpan`
   from `deckLayouts.<layout>.positions`. Calculate the position's center by
   adding `xSpan / 2` to the X coordinate.
4. **Check if the position's center is outside the X range of the pod.** Depending on
   whether the pod is moving to the position for pipetting or for gripping, and
   whether the position includes offsets, there may be additional offsets that
   impact this result. However, if the position is well outside the range, do
   not use it with this pod.
5. **Repeat for Y** where the position is in the frontmost or rearmost row, against `min.y` / `max.y`.

If a position fails the check there are exactly two fixes: move the labware to a position inside
that pod's envelope, or assign the step to the pod whose envelope contains the position. Changing
what else is on the deck is not a fix.

## Position reach table - Pipetting

In the nominal case, the following table shows which deck columns are reachable
by each pod on each instrument for pipetting. To determine whether a specific
position on a deck is reachable, in the instrument settings export, check the
position's `group`'s `column` value.

This table shows positioning for standard static ALPs (Static1x1, Static1x3,
Static1x5, TipLoad1x1, Static1x1_i3, Short1x1_i3). Other ALPs define their
locations differently and these values may not apply; check the exact framed
coordinates instead.

| Instrument | Pod Type | Leftmost Column | Rightmost Column |
|------------|----------|-----------------|------------------|
| i3 | Fixed-8 | 3 (C) | 22 (V) |
| i5 | Multichannel | 6 (F) | 34 (AH) |
| i5 | Span-8 | 6 (F) | 34 (AH) |
| i7 Single-Arm | Multichannel | 6 (F) | 62 (BJ) |
| i7 Single-Arm | Span-8 | 6 (F) | 62 (BJ) |
| i7 Hybrid | Left Multichannel | 6 (F) | 48 (AV) |
| i7 Hybrid | Right Span-8 | 27 (AA) | 62 (BJ) |
| i7 Dual MC | Left Multichannel | 6 (F) | 41 (AO) |
| i7 Dual MC | Right Multichannel | 27 (AA) | 62 (BJ) |

## Position reach table - Gripping

In the nominal case, the following table shows which deck columns are reachable
by each pod on the instrument for gripping. To determine the column of a
specific position, in the instrument settings export, check the
position's `group`'s `column` value.

This table shows positioning for standard static ALPs (Static1x1, Static1x3,
Static1x5, TipLoad1x1, Static1x1_i3, Short1x1_i3). Other ALPs define their
locations differently and these values may not apply; check the exact framed
coordinates instead.

Note that for the Multichannel and Span-8 arm, the leftmost column can only be
accessed with the "A1 away from the gripper" grip side, and the rightmost only
with "A1 near the gripper".

Note that ALPs in positions left of column F are typically associated with
integrations and are only accessible for gripping with "A1 away from the
gripper". The exact extents of reachability on the left will depend on the
particular instrument's calibrated values.

| Instrument | Pod Type | Leftmost Column | Rightmost Column |
|------------|----------|-----------------|------------------|
| i3 | Fixed-8 | 3 (C) | 22 (V) |
| i5 | Multichannel | -2 (-C) | 34 (AH) |
| i5 | Span-8 | -2 (-C) | 40 (AN) |
| i7 Single-Arm | Multichannel | -2 (-C) | 62 (BJ) |
| i7 Single-Arm | Span-8 | -2 (-C) | 68 (BP) |
| i7 Hybrid | Left Multichannel | -2 (-C) | 48 (AV) |
| i7 Hybrid | Right Span-8 | 20 (T) | 68 (BP) |
| i7 Dual MC | Left Multichannel | -2 (-C) | 41 (AO) |
| i7 Dual MC | Right Multichannel | 20 (T) | 62 (BJ) |
