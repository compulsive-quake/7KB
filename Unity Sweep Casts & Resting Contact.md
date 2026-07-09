# Unity Sweep Casts & Resting Contact

For custom (non-Rigidbody) flight/movement physics that sweeps a box each
frame (`Physics.BoxCastAll` from current position along the frame's velocity)
and stops/bounces on hit:

## Gotcha: zero-distance "initial overlap" hits freeze a resting object

PhysX reports colliders the swept shape **already overlaps at the sweep
start** as hits with `distance = 0`, `point = Vector3.zero`, and
`normal = -castDirection`. Two consequences:

1. **Garbage contact data.** Reflecting velocity off that normal, or building
   an impact at that point, uses meaningless values (the "hit point" is the
   world origin).
2. **Takeoff freeze.** An object that settles into resting contact with the
   ground sees the floor as a zero-distance hit on *every* sweep — including
   one pointing straight **up**. If the collision response zeroes/reflects
   velocity on any hit, the object can never leave the ground: thrust
   integrates into velocity, the sweep "hits", velocity is cancelled, repeat.
   Symptom: motors/input clearly responding, object pinned in place.

Fix: skip hits with `distance <= 0` in movement sweeps (genuine approaches
always report a positive distance), and/or keep grounded objects parked on a
small virtual floor above the surface so they never enter resting contact.
The FPV mod does both — see `BoxCastWorld` and the pre-takeoff pad floor in
`FPV/src/FPVDroneManager.cs`.

History: the FPV drone originally rested on a virtual pad floor
(`_launchStartWorldPos.y`) and never touched terrain. A commit removed the
clamp (it blocked diving below launch height off a cliff) and routed
pre-liftoff ground contact through the normal cold-bounce/rest path — which
hit this freeze and made the drone unable to take off at all. The repair kept
the floor but releases it on horizontal clearance from the pad as well as
vertical, and added the zero-distance skip.
