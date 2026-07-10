# Moving Platforms & Riding Entities

Part of the [7DTD Modding Knowledgebase](README.md). How to make a runtime-moved
collider (elevator car, moving platform — see
[Moving Blocks at Runtime](Moving%20Blocks%20at%20Runtime%20(SetBlocksRPC).md))
carry players, zombies and vehicles *without* blocking their own movement.
Everything verified against decompiled V2.x (2026-07, Elevator mod).

---

## The wrong way: per-frame `SetPosition` on riders

Teleporting every entity in the car volume each frame "works" but stomps
movement:

- **`EntityPlayerLocal.SetPosition` → `vp_FPController.SetPosition`**, which
  resets `m_PrevPos` / `m_SmoothPosition` / `m_FixedPosition` and calls
  `Physics.SyncTransforms()` — every frame. The UFPS controller's own motor
  fights it; riding feels frozen/jittery and input barely registers.
- It drags bystanders: anything whose AABB clips the moving volume (someone
  half-on a landing the car passes) gets yanked.

## Layer table (TagManager, V2.x)

Read from `7DaysToDie_Data\data.unity3d` → TagManager via UnityPy
(`obj.read_typetree()['layers']`; UnityPy must run under the Python that has
it installed — `C:/Python311/python.exe` on this machine):

| # | Layer | # | Layer |
|---|---|---|---|
| 0 | Default | 16 | TerrainCollision |
| 1 | TransparentFX | 17 | Physics Dead |
| 2 | Ignore Raycast | 18 | Grass |
| 3 | CC RemotePlayer Physics | 19 | LargeEntityBlocker |
| 4 | Water | 20 | CC Local Physics |
| 5 | UI | 21 | Physics |
| 7 | LightLocal | 22 | UnderwaterEffect |
| 8 | NoShadow | 23 | Trees |
| 9 | Background Image | 24 | LocalPlayer |
| 10 | HoldingItem | 25 | NotInReflections |
| 11 | RenderInTexture | 26 | ReflectionsOnly |
| 12 | NGUI | 28 | Terrain |
| 13 | Items | 29 | Wires |
| 14 | NoCollision | 30 | Glass |
| 15 | CC Physics | 31 | Volumes |

Collision-matrix facts that matter for platforms (PhysicsManager
`m_LayerCollisionMatrix`, same dump):

- **13 "Items"** (vanilla falling-block layer) collides with vehicles (21) and
  items, but with **no character controller** — not local player (20), not
  zombies (15), not remote players (3). A proxy on 13 is *not ground* to
  anyone walking; riders stay on it only if you teleport them.
- **16 "TerrainCollision"** (chunk colliders) collides with **every mover**:
  20, 15, 3, 21, 13, 17, 24. Put a rideable platform's colliders here and
  players/zombies/vehicles/items treat it exactly like world geometry — they
  can step on and off at will.
- The `vp_FPController` ground probe (SphereCast mask `1084850184`) hits
  layers {3, 15, 16, 19, 21, 23, 30} only. A platform outside that set never
  registers as ground for the local player.

## Move everything per RENDER frame, not per fixed step

Rule learned the hard way (Elevator, 2026-07): **any part of the
platform-rider system that advances at fixed-step rate (~50 Hz) stair-steps
~5 cm per tick against 100+ fps rendering and reads as shake/blur** — the
camera judders against the world and the car mesh. This rules out the two
"proper physics" approaches:

- `rb.MovePosition` from `FixedUpdate` + interpolation renders the platform
  smoothly, but every *rider* still moves in fixed steps.
- The engine's UFPS moving-platform code (below) tracks the player only in
  `FixedUpdate` **and deliberately snaps vertical smoothing while the
  platform's y moves** (`SmoothMove`: `m_SmoothPosition.y =
  Transform.position.y` when `m_Platform` y changed) — vertical judder is by
  design there; only the smoothing patch-out *and* per-frame platform motion
  would fix it, and the smoothing lerp is far too slow (τ ≈ 1 s) to lean on.

The butter-smooth recipe: advance the glide in `Update`, set the proxy root's
`transform.position` every frame (`world - Origin.position`, which also
absorbs Origin repositions for free), and carry riders per frame as below.
Driven vehicles still ride fine on a per-frame-teleported kinematic collider:
their suspension re-solves each physics step and absorbs the missing contact
velocity (bounce amplitude ≈ g·dt² — invisible).

## Local player: state-preserving controller shift (per frame)

Two dead ends first:

- **`EntityPlayerLocal.SetPosition` → `vp_FPController.SetPosition`** resets
  `m_PrevPos`/`m_SmoothPosition`/`m_FixedPosition` + `Physics.SyncTransforms()`
  every call — per-frame use stomps input and feels frozen.
- **The dormant UFPS platform support** (`m_Platform`, `UpdatePlatformMove`,
  jump-off velocity inheritance) is functional and revivable — vanilla's
  trigger `m_GroundHit...layer == 28` is dead because the ground probe mask
  excludes 28 — via a postfix on `vp_FPController.UpdateCollisions` setting
  `__instance.m_Platform` when grounded on your proxy. It gives free walking
  and jumping, **but stair-steps the camera at fixed-step rate** (see above).
  Works; not smooth. Don't ship it for a slow vertical platform.

What ships: shift the controller and **all of its cached positions** by the
platform's per-frame delta, in one piece. A uniform translation is invisible
to the motor — walking/sprint/jump input keeps working, nothing is reset, no
fall impacts, and the camera glides at render rate:

```csharp
vp_FPController c = player.vp_FPController;          // publicized property
if (c == null || !c.m_Grounded) return;              // airborne: don't carry
Collider ground = c.m_GroundHit.collider;
if (ground == null || !ground.transform.IsChildOf(proxyRoot.transform)) return;
Vector3 shift = new Vector3(0f, dy, 0f);
c.transform.position += shift;
c.m_SmoothPosition   += shift;
c.m_FixedPosition    += shift;   // FixedUpdate rewinds transform to this — must move too
c.m_PrevPos          += shift;   // used for velocity derivation
player.fallDistance = 0f;
```

Grounding stays stable because the shift preserves relative geometry exactly:
the fixed-step ground probe sees the same arrangement every time. Jumping
clears `m_Grounded` next fixed step, which ends the carry cleanly.

## Vehicles

Driven vehicles are dynamic rigidbodies (layer 21) — they ride the proxy
colliders through their suspension with no help (see per-frame note above).

**Scripted-locomotion vehicles (Route B, e.g. Oppressor) are a platform-side
blind spot**: their soft-floor logic reads `World.GetHeight` (voxel
heightmap), which reports the floor gone while the platform's blocks ride as
a proxy — the vehicle sinks through and snaps back up when the blocks land.
The manual carry can't save it: the vehicle's own per-tick `SetPosition`
overrides the carry every tick. Fix belongs in the *vehicle*: make its floor
sensing collider-aware (max of voxel floor and a downward SphereCast on
layer 16) — see the "Soft floor" section of [Vehicles](Vehicles.md).

Parked vehicles are a different beast: `isKinematic = true` and they settle
through `Entity.aabbEntityCollision` — **voxel-AABB collision that ignores
Unity colliders entirely**. A parked car on your platform must be carried
manually (vertical delta + clamp `motion.y` up to 0 + `fallDistance = 0`).

## Zombies / animals / remote players

Entity locomotion dispatch (`Entity.entityCollision`): entities **with** a
`m_characterController` collide via PhysX (`ccEntityCollision` →
`CharacterControllerKinematic`, a wrapper around the KinematicCharacterController
package); entities without one use voxel AABB (`aabbEntityCollision`).
Zombie/animal CCs are on layer 15, so a layer-16 platform is solid ground to
them — but the game constructs KCC motors with
`InteractiveRigidbodyHandling = false`, so KCC's own moving-platform ride is
off. Descent works by gravity alone; for ascent (and voxel-collision
entities) apply the per-frame vertical delta manually via
`SetPosition(pos + dy)` — additive, so their own AI movement keeps working.

Filter who you carry — probe straight down from the entity and require the
hit to be *your* platform, so real floors between entity and platform win:

```csharp
Vector3 origin = rider.position - Origin.position + Vector3.up * 0.4f;
bool riding = Physics.SphereCast(origin, 0.3f, Vector3.down, out var hit, 2f,
                  1 << 16) && hit.transform.IsChildOf(proxyRoot.transform);
```

Skip `EntityPlayerLocal` (platform patch above), skip
`rider.AttachedToEntity != null` (seated — moves with the mount), skip
`EntityVehicle` with `!vehicleRB.isKinematic` (driven — solver carries it).

## Block-targeting rays now hit the platform (layer 16)

`Voxel.raycastNew` masks include 16 but exclude 13 — that's why vanilla
falling blocks hide on 13. Resolution is **tag-driven**, not layer-driven:
an untagged collider hit returns a generic non-block hit (`bHitValid`,
`voxelRayHitInfo.transform` set), and `T_Block`-tagged clones resolve via
`FindMasterBlockForEntityModelBlock` with a graceful fallback when the voxel
underneath is air. Net effect: bullets/melee stop on the moving car without
damaging anything, no NREs observed in the paths read. If a platform must be
transparent to targeting, layer 13 is the hiding spot — but then nothing can
walk on it.
