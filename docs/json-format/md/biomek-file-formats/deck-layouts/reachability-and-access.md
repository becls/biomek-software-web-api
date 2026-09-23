# Reachability and Access

Not every deck position can be used by every pod, and a position being reachable does not
mean every well of the labware on it can be pipetted, or that the labware can be moved off
it. A method that enqueues cleanly on one instrument can fail on another with the same deck
because of the pods installed, where a position sits in the pod's travel, or what else is on
the deck at the time.

**This page is a troubleshooting aid, not a specification.** The one number it does publish is
each pod's **X-axis travel envelope**, because that is a hard limit the export records and the
single most common reason a position is unreachable — see "Reading a pod's travel envelope"
below and the worked i7 figures in
[Pod Reach Envelopes (i7 worked example)](./pod-reach-envelopes.md). Everything else (per-position framing, the
live two-pod interaction, what else is on the deck) still depends on your chassis and
calibration; use this page to work out *which* constraint you have hit and what to change, and
confirm the specifics in the Deck Editor and Manual Control.

For what an ALP is and which instruments each one suits, see
[ALP Catalog](./catalog-alps.md). For the JSON fields mentioned here, see
[Deck Layouts Format Specification](./deck-layouts-format-spec.md).

## Two different questions

Reach is not one test. Biomek answers two questions separately, and a position can pass one
and fail the other:

| Question | Applies to | Typical failure |
|---|---|---|
| Can the pod get its tips over this position to pipette? | Transfer, Combine, and other pipetting steps | `The <pod> cannot reach the deck location "<position>".` |
| Can the pod's gripper get to this position, holding this labware, on this grip side? | Move Labware, and steps that fetch tip boxes | `<pod> cannot grip labware at <position>.` / `<pod> cannot use this grip …` |

The gripper question additionally has a coarse form and an exact form. The coarse form asks
whether the move is geometrically possible on a **bare deck**; the exact form asks whether it
is possible **given what is on the deck right now**. A move that the Deck Editor accepts can
still fail at run time once neighboring positions are occupied.

## Symptom to likely cause

The message you see often does not name the constraint you actually hit — most notably, an
unreachable *column* reports a path-planning failure, not a reach failure.

| Message you see | What it usually means | What to try |
|---|---|---|
| `The <pod> cannot reach the deck location "<position>".` | The position as a whole is outside that pod's travel. | Move the labware to a more central position, or use the other pod. |
| `Unable to find a path …` | Often a *partial* reach problem: the position is reachable but the specific wells, columns, or approach are not — or the route is blocked. **Read the rest of the message**: if it continues `… destination <axis> <n> cm is outside of travel range, which is between <min> and <max>`, it is a hard axis limit, not an obstruction. | For a travel-range clause, move the labware to a position within range — clearing neighbors cannot help. Otherwise try a different column or well selection, clear tall neighbors, or check whether a second pod is limiting travel. |
| `Limit violation: Destination (Z:<n>) is outside the Z limits (<min>, <max>).` | A hard **pod Z-axis** limit, reported in cm of deck height. The move would put the pod above its ceiling or — far more often — below its floor. On the i3 this usually means a low-seated position: short-ALP positions sit much closer to the deck than standard positions, and gripping or loading tips there can drive the pod under its Z floor. | Use a standard-height position instead of a short one, or shorter labware. Clearing neighbors cannot help — this is an axis limit, not an obstruction. See "The pod's Z travel band" below. |
| `Unable to find a path to move the pipettor to a safe Z height: … would cause a collision with obstacle <position>: <class>.` | A genuine **obstruction**, not an axis limit: a tall item — most often a tip box — sits in the pod's travel corridor. An obstacle named `<position>: TipHubs` is not a labware class: it is the stub of tips standing proud of a tip box's surface, modeled separately from the box body, so the blocker is the *present tips* at that position rather than the box itself. The blocker sits between two positions and stops the safe-Z move (frequently seen prefixed `Error cleaning tips for transfer from <position>:`). | Relocate the tall obstacle named in the message out of the corridor between the source and destination positions; this one *is* fixed by clearing the neighbor, unlike a travel-range clause. |
| `Can't lower Pod<n> to tip load height (…) over <position> because it would cause a collision: …` (also `…to tip preload height…` / `Lowering the pod to tip load height would cause a collision.`) | A collision while descending to pick up tips — the head cannot reach tip-load height over that tip position because a neighbor obstructs it. Distinct from the safe-Z pipetting collision above. | Clear the obstructing neighbor, or move the tip box to a tip-load position with clearance. |
| `Unable to move <pod> gripper GZ axis to <n> to grab plate at <position>: <part> of <pod> gripper interferes with position <neighbor>.` (also `… to put labware at …`) | A **gripper-body collision with a neighboring deck position**, not with the labware you are moving. The named part — `lowerhand`, `upperhand`, or a finger portion — sweeps into the adjacent position as the gripper descends. Positions immediately beside a wash or trash station are the common case. Read the blocker's wording closely: `position <name>` is the neighboring position itself, so clearing what sits on it will not help; a blocker phrased `obstacle <name> on position <name>` *is* an item you can remove. | Move the labware to a position that is not adjacent to the named neighbor, or grip from the other side. |
| `The selected probes cannot reach the given section of the reservoir.` | Span-8 only. The chosen probes, at their spacing, do not fit inside that reservoir section. | Select fewer probes, choose different probes, or target a wider section. |
| `Cannot access labware of type <class> with <pod>.` | The head's geometry cannot address that labware's wells at all — not a position problem. | Use a pod whose head suits the labware; low-density plates generally need Span-8. |
| `<pod> cannot use this grip …` | No single grip side works at **both** ends of the move. | Try the other grip side; if neither works, change one end's position. |
| `Cannot pick up labware that would extend … before hitting the mandrels.` | **Fixed-8 only.** The labware or stack is too tall for the integrated pod gripper at the requested grip height. | Use a higher grip offset, or move fewer plates at a time. |
| `Cannot pick up labware when tips are loaded.` | Tips are on the pod. | Unload tips before the move. |
| `<pod> is unable to find a tip discard location.` (Span-8) and `<pod> cannot find a location to discard tips.` (Multichannel) | No deck position carrying the `can Discard Tips` characteristic is reachable **by that pod**. The search filters on the characteristic *and* pod reach, so a trash the other pod reaches can be out of range here — see "Two pods on one bridge" below. | Place a trash inside that pod's envelope, or assign the unload to the pod that can reach one. Neither step names a destination, so this is a deck fix rather than a parameter change. |
| `Unable to find a trash position to discard tips.` (Fixed-8) | No deck position carries `can Discard Tips` at all. This path tests the characteristic alone with no reachability check, and never names a pod. | Add a trash position, or set the step's tip destination to return tips to the box they came from — see [Fixed-8 Unload Tips](../../step-documentation/40-Fixed-8-Unload-Tips.md). |
| `<position> does not allow gripping.` | The position is flagged against gripping regardless of geometry. | Use a different position; see the characteristics note below. |

## What limits reach

### Pod travel and the position center

A pod's position-level reach test is evaluated at the **center of the position**, so it is
all-or-nothing: it tells you whether the pod can address that position, not whether every
well of the labware there is reachable. Trash and tip-disposal positions are the exception —
they are evaluated at their tip-discard point rather than their center, because they are
usually placed at the end of pod travel where the center would not be reachable.

### The pod's Z travel band

Reach is limited vertically too, and the band is narrower than most authors expect. Each pod
publishes it in the instrument-settings export as `podSettings.<pod>.settings.min.z` and
`max.z`, in cm of deck height — on an i3, Pod1 runs nominally from **9.472** to **26.286**. A move outside it
fails with the `Limit violation: Destination (Z:<n>) is outside the Z limits (…)` message above.

**The floor is the one that bites.** A position whose ALP seats labware close to the deck leaves the
pod nowhere to go: on the i3, short-ALP positions seat labware about 2.8 cm above the deck
against roughly 8.6 cm at standard positions, and gripping labware or loading tips there can put the pod
below 9.472. This is why the short positions carry a characteristic that disallows manual
teaching — the Fixed-8 pod does not have the Z travel to under-grip labware directly off them, so
they are framed by auto-teach or with a framing block instead.

**Do not try to predict this by adding heights together.** The number checked against the band is a
pod-axis coordinate, not "seat height plus labware height", and it is computed differently for
different operations: a tip load works from the ALP seat, the tip *box* class height, and a
press-on offset, while a gripper move works from the ALP seat, the pod's own gripper Z offset, the
stacking offset at the position, and the labware's grip offset — which is negative for an under-grip.
Arithmetic on the tip class's own height does not enter either one. Treat the published band as a
diagnostic for reading the error, and confirm a marginal position by trying it.

### Individual columns and wells

Because the position-level test uses the center, a position near the edge of travel can pass
it and still have columns the pod cannot address. On a compact chassis, labware at the
far-left position may be usable for its rightmost column and not its leftmost. This surfaces
as a path-planning failure rather than a reach message, so if a transfer fails only for
certain column or well selections, suspect edge-of-travel geometry before suspecting the
step. Narrowing the selection to the columns nearer the deck center is the quickest test.

### Positions that are simply out of range

Reach is limited front-to-back as well as side-to-side, and a deck can define positions that a
given pod cannot reach **at all**. This is the case worth ruling out first, because it looks like
every other reach problem and responds to none of the usual remedies.

The tell is in the message. A hard axis limit reports the axis, the requested coordinate and the
pod's actual range:

```
Unable to find a path for <pod> pipettor to approach position <position>:
Specified pipettor destination Y <n> cm is outside of travel range,
which is between <min> cm and <max> cm.
```

When you see that, stop looking for a collision. Emptying neighboring positions, changing the load
order, rearranging tips and re-framing will all fail, because nothing about the deck's *contents* is
being complained about. The position is out of the pod's reach and the labware has to move.

For how to read the pod name, the requested coordinate and the two limit values in that message
against the pod's published envelope — and a real example of each axis form — see
[Pod Reach Envelopes (i7 worked example)](./pod-reach-envelopes.md#the-travel-range-error-reading-the-two-numbers) §"The travel-range error: reading the two
numbers".

Back-row positions are the usual offenders — on a multichannel layout the rearmost row of positions
(including the rearmost tip-load position) can sit beyond the pod's Y travel once the head's own
depth is added to the position's coordinate. They are listed in the deck layout exactly like every
other position, with nothing to mark them out, so a method that spreads labware evenly across the
published positions can land on one without warning. If you are allocating several tip boxes or
plates across a row, prefer positions nearer the front and confirm the rearmost one in Manual
Control before committing to it.

The front row positions are common offenders on a Span-8 configuration. Probe 1
cannot reach wells at the front of the front row. Probes 7-8 should be used
instead if access is required, or labware placed farther back.

Note also that being out of range only matters when a pod actually has to go there. A position that
is merely *named* by a non-pipetting step — a Pause that takes a deck position as a scheduling
resource, say — involves no pod motion and is not subject to these limits, so an unreachable
position can appear in a working method without ever failing.

### Reading a pod's travel envelope

The travel range in that message is not a mystery number — the instrument records it, per pod.
In a Biomek Instrument Settings export each pod carries its X (and Y, Z, …) limits at
`podSettings.<pod>.settings.min.x` / `.max.x`, and the same values appear in human-readable form
in `podSettings.<pod>.asText` under *Axis Limit Settings* — which is also what the Pod Settings
dialog shows in the software. Read those to know, before you enqueue, how far each pod can go.

The catch is the coordinate frame: those limits are **pipettor-axis** values, and a deck
position's `x1` is offset from them by a fixed per-pod amount (head and mandrel geometry) that
the export does not publish. So the envelope reliably tells you the *shape* of the constraint —
the right-hand pod loses the left of the deck, the left-hand pod loses the right — but the exact
`x1` cut-off has to be confirmed in Manual Control unless you can anchor the offset from an
observed reach failure. Worked i7 envelopes
are in [Pod Reach Envelopes (i7 worked example)](./pod-reach-envelopes.md).

### Tip loading: capability is not reachability

The `characteristics` keys `multichannel Can Load Tips` and `span Can Load Tips` advertise which
pod family a position is *built* to load tips for — a **capability**, not a promise that the pod
can currently **reach** it. The two are not symmetric, and confusing them is a common trap:

- **Multichannel tip boxes are confined to the tip-load positions.** Only the `tL*` positions
  carry `multichannel Can Load Tips`; a plain `p*` position never does. An MC tip box has nowhere
  else to go.
- **Span-8 tip boxes can relocate anywhere.** Every plain `p*` position carries `span Can Load
  Tips`, so a Span-8 tip box is not tied to the `tL*` column — this is the escape hatch when the
  `tL*` positions do not work for the Span-8.
- **The trap:** the `tL*` positions *also* carry `span Can Load Tips`, so it looks like Span-8
  tip loading works there. On a two-pod instrument where the Span-8 is the right-hand pod (the i7
  Hybrid), the `tL*` column sits at the far left, below the Span-8's X minimum — the Span-8
  cannot reach it, capability flag notwithstanding. Put the Span-8 tip box on a `p*` position
  inside the Span-8's reach instead. The i7 Hybrid figures are in
  [Pod Reach Envelopes (i7 worked example)](./pod-reach-envelopes.md).

### Grip side

For the Multichannel and Span-8 pods, labware can be gripped with the A1 corner **near** or **away
from** the pod. A grip side is only usable if it works at
**both** the source and the destination of the move. This is the common surprise: each end
may be fine on its own, but with different sides, leaving no side that serves the whole move.

Grip side is not universal. A Fixed-8 pod does not offer a grip-side choice at all.

### Labware and stack height

Gripping is limited from above, not below. On a **Fixed-8** pod (i3), where the gripper is
integrated directly beneath the mandrels, a stack that would extend too far up from the bottom of
the gripper fingers collides with the mandrels, and the check accounts for the grip offset: an
**undergrip** brings the labware closer to the mandrels, while a **high grip** buys clearance.
Labware that fails to move as a stack may move when gripped higher.

On **Span-8** and **Multichannel** pods (i5/i7) the gripper is a *separate rotating gripper* that
swings the labware clear of the pipettor, so height is not bounded by mandrel clearance. There the
move is limited instead by the gripper's safe-Z / travel ceiling and by collision against deck
obstacles and the pipettor body — gripping higher can still help, but through that clearance model
rather than a fixed distance to the mandrels.

Some labware cannot be undergripped at all — where the external walls taper, there is nothing
for the gripper's toe to get under, and the grip offset must be raised to catch the skirt.
Class descriptions in the labware catalog call this out where it applies.

Height also constrains from the other direction on a compact chassis. The same low profile that
lets a Fixed-8 pod reach *into* labware to pipette can leave it no room to reach *below* that
labware for an undergrip. See "The pod's Z travel band" earlier on this page for the limit behind
this and the message it raises.

### Two pods on one bridge

When two pods share a bridge, neither gets its full travel: each pod's range is narrowed by
where the other one is, so a position reachable on a single-pod instrument may be unreachable
on a two-pod one with the same deck. Which end is clipped depends on which side the other pod
is on. If a position works when a method is run one way and not another, or works after a
step that parks the other pod, this is the likely cause — the answer depends on pod state,
not on the deck alone.

Refer to [Pod Reach Envelopes (i7 worked example)](./pod-reach-envelopes.md).

### Head geometry versus labware layout

Distinct from position reach: a head may be unable to address a labware class's wells at all,
because the well spacing does not match the head's channel pitch. Low-density plates — 6-,
12-, 24-well culture plates — are the usual case. This fails on the labware class, not the
position, so moving the plate elsewhere does not help.

### Positions that block access outright

Some positions refuse access for reasons unrelated to geometry: a position may be flagged to
disallow gripping, hold permanent labware that method authors do not place or move, or be a
trash device. These appear as `characteristics` keys on the position in an instrument-settings
export — see the `characteristics` row in
[Deck Layouts Format Specification](./deck-layouts-format-spec.md). A position advertises which pod
families can load tips from it the same way.

## Depth is a separate axis

"Cannot reach" is also used for **depth** — a tip that is not long enough to get to the
requested height in a deep well without the pod striking the labware, or a technique whose
minimum-height setting holds the tip above the target. These report their own messages,
naming tip length and depth in cm, and are properties of the tip class, labware class, and
technique rather than of the deck position. If a failure names a depth or a tip length,
it is not a deck-reachability problem.

## Narrowing it down

1. Does it fail for the position, or only for some wells or columns? Position-wide points at
   pod travel; selection-specific points at edge-of-travel geometry.
2. Does it fail for pipetting, gripping, or both? They are separate tests with separate
   messages.
3. For a failed move, does either grip side work on its own at each end? If so, the move
   needs one side that serves both.
4. Does it fail only with both pods in play, or only after a particular step? That points at
   the two-pod travel limit rather than the deck.
5. Does the message name a labware class rather than a position? That is head-versus-labware
   geometry, and relocating will not help.

## Notes

- Reach depends on framing. A position that has not been framed for a given pod, or has been
  framed to different offsets, will not behave like the same position on another instrument.
- Reach is evaluated per pod. On a two-pod instrument, "the deck" does not have one reach
  answer; each pod has its own, and each is affected by the other's position.
- The sample decks in [samples/](./samples/) show where positions sit on each default layout,
  which is often enough to spot an edge-of-travel position by eye.
