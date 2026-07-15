# Elevator Mod — Floor Anchor Model

How the Elevator mod (`7days/Elevator`) tracks floors and the car, and the
desync trap fixed on 2026-07-10.

## The model

- `BoundsMin` is **both** the ground floor's world position **and** the base
  for the car's `CurrentOffset` (car world Y = `BoundsMin.y + CurrentOffset`).
- Floor offsets are stored as per-floor `Rise` gaps, chained and expressed
  relative to the ground floor (`OffsetOfFloor(GroundFloor) == 0`).
- `CurrentOffset` is physical truth: the car's blocks are always at
  `BoundsMin + CurrentOffset` by construction (persisted every glide step).

## The trap (fixed)

`ElevatorBoundsEditor.Start` seeds the edit box at the **car's current
position** (`BoundsMin + CurrentOffset`), so a naive `Commit` that does
`BoundsMin = box; CurrentOffset = 0` re-anchors floor 1 wherever the car
happens to be parked. Re-editing bounds while parked upstairs silently
shifted the whole floor stack up — panel says "floor 1" on the building's
second story, and the true bottom story becomes unreachable ("already at
floor 1").

Rule: **re-editing bounds only repositions the car**; keep the ground Y
(`CurrentOffset = box.y - BoundsMin.y; BoundsMin.y` unchanged). Only
first-time setup pins ground at the box.

## Moving the ground floor

The ground floor is movable like any other, but it is the anchor, so moving
it is a pure relabel — no blocks move:

- apply the same neighbor `Rise` updates as any floor move, **plus**
- `BoundsMin.y += delta` and `CurrentOffset -= delta`

Net effect: every other floor and the parked car keep their world heights;
only the reference plane moves. Guard with the bounds editor's world-height
clamp (`1 .. 250 - size.y`) and refuse while `ElevatorMover.Active`
(the mover writes `CurrentOffset` concurrently).

## UX: the anchor row must show live feedback (added 2026-07-10)

Because the ground row's *relative* height is 0 by definition, showing that
offset makes the +/- steppers look dead — the number never changes no matter
how many times you click (the user reported "can't adjust floor 1 rise after
creation" while the code path worked fine). Fix: the ground row's Height
field shows the anchor's **world Y** (`"y" + BoundsMin.y`) instead of the
frozen 0, and every refused adjustment (world-height clamp, 1..MaxRise gap
limits, bounds not set) shows a `GameManager.ShowTooltip` explaining why
instead of a silent `return`.

General UI rule: a stepper that silently no-ops or edits a value the row
doesn't display reads as broken. Surface the value that actually changes,
and tooltip every refusal.

## General lesson

When a data model stores positions relative to an editable anchor, every
anchor edit must explicitly choose which side stays fixed in the world —
and compensate the other side. Letting the anchor rebase implicitly is how
"the panel says floor 1 but I'm on floor 2" bugs happen.
