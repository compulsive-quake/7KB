# Homing Missiles (lock-on + scripted flight)

GTA-style vehicle homing rockets built entirely in code — no XML projectile, no
asset bundle. Reference implementation: `Oppressor/src/OppressorRockets.cs`
(lock-on system + IMGUI detection box + scripted missile), wired from
`EntityOppressor.OnUpdateLive`. The turret-style ancestor is
`RocketTurret/src/RocketTurretLaunchManager.cs` — **note its `ExplosionServer`
call and flat `Explosion.*` props are 2.x-era and silently broken on 3.0.0**
(see [Runtime Explosions (ExplosionData)](Runtime%20Explosions%20(ExplosionData).md)).

## Scripted missile (MonoBehaviour, no Entity)

A plain GameObject + MonoBehaviour, keeping its own **world-space** position and
stamping `transform.position = worldPos - Origin.position` each frame (origin
reposition — see [Entity Positions](Entity%20Positions.md)).

Per frame:
1. Speed ramp (`MoveTowards` toward max) — launch feel.
2. Homing: `dir = Vector3.RotateTowards(dir, (aimPoint - worldPos).normalized,
   turnRateRad * dt, 0f)`. Capped turn rate (~165°/s) is what makes fast
   crossing targets dodgeable. Aim point = `target.position + up *
   max(0.8, target.height * 0.55)`.
3. Proximity fuse: detonate within ~2 m of the aim point (covers frame-step
   overshoot at 50 m/s).
4. Collision: `Voxel.Raycast(world, new Ray(worldPos, dir), step + 0.35f,
   -538750997, 8, 0f)` — world coords in, `Voxel.voxelRayHitInfo.hit.pos`
   world coords out; this mask hits terrain, blocks, AND entity colliders, so
   direct hits detonate too. Pull the blast back along `dir` ~0.12 m so the
   explosion sphere isn't buried in the surface.
5. Arm delay (~0.1 s of no collision checks) so the rocket clears the launching
   vehicle's own colliders.
6. Lifetime cap → detonate airborne (reads better than despawning).

Detonation = 3.0.0-style `ExplosionData` (nested `"Explosion"` class via
`props.SetValue("Explosion", key, value)`) + `ExplosionServer(worldPos,
World.worldToBlockPos(worldPos), Quaternion.identity, data, ownerEntityId, 0f,
false, null)`. Frag-style tuning (high `EntityDamage`/`RadiusEntities`, low
block damage) keeps horde-strafing from cratering bases/POIs.

## Vanilla assets to reuse (no bundle needed)

- **Rocket visual**: `@:Other/Items/Weapons/Ranged/RocketLauncher/rocketPrefab.prefab`
  via `DataLoader.LoadAsset<GameObject>`. Nose is **+Z** (matches
  `Quaternion.LookRotation(velocity)`). Must: destroy MonoBehaviours whose type
  name contains `Projectile`/`Rocket`/`Mover` (they'd drive their own flight),
  destroy Rigidbodies, disable colliders, then `SetActive(true)` every child
  (smoke/flame ship inactive; the launcher item normally activates them) and
  `Play()` all particle systems.
- **Launch bang**: sounds.xml node `m136_fire`
  (`GameManager.PlaySoundAtPositionServer(pos, "m136_fire",
  AudioRolloffMode.Logarithmic, 80, 1f)` — 3.0 has a trailing `_volumeScale`).
- **Flight loop**: the raw clip is loadable directly —
  `DataLoader.LoadAsset<AudioClip>("@:Sounds/Weapons/Ranged/M136/m136_rocket_lp.wav")`
  onto your own looping 3D AudioSource (dopplerLevel ~0.6 gives the flyby
  whoosh). Loading `.wav` assets by `@:` path works for any sounds.xml clip.
- **Explosion visual**: `ParticleIndex 5` = vanilla `ammoRocketHE` blast
  (vanilla HE profile for reference: EntityDamage 420, RadiusEntities 4,
  BlockDamage 2500, RadiusBlocks 5).

## Lock-on system

- **Acquisition**: candidates from `world.GetEntitiesInBounds(typeof(EntityAlive),
  bounds, list)`; filter class (`EntityEnemy` subtree covers zombies —
  `EntityZombie : EntityHuman : EntityEnemy` — and hostile animals via
  `EntityEnemyAnimal : EntityEnemy`; add `EntityAnimal` for wildlife; players/
  NPCs/traders excluded by not being either). Pick the smallest angle from
  `rider.cameraTransform.forward` inside an acquire cone (~15°), range ~90 m.
- **Sticky hold**: once locked, keep the target while it stays in a *wider*
  cone (~26°) and range ×1.15, and give cone/LOS failures a ~0.3 s grace
  window before dropping — otherwise the box flickers on every fence post.
- **LOS from the vehicle, not the camera**: a 3rd-person camera ray passes
  through the vehicle's own `E_Vehicle` collider (and the rider) every frame.
  Cast from vehicle center offset ~2.4 m along the ray (clears a bike-sized
  collider's half-diagonal), stop ~0.7 m short of the target so its own
  collider isn't counted as an occluder.
- **Lock progress**: same target held → `progress += dt / 0.85s`; new target
  resets. `progress >= 1` = locked.
- Run all of this **per frame in a persistent MonoBehaviour**, not in the
  entity's 20 Hz tick — `Input.GetMouseButtonDown` edges land on frames the
  entity doesn't tick, and the HUD box needs per-frame tracking. The vehicle
  registers itself each tick (`RegisterActive(bike, rider)` + timestamp) and
  the runtime treats a stale timestamp (>0.3 s) as dismounted — no explicit
  teardown paths needed.

## Detection box HUD (IMGUI)

Project the 8 corners of `position ± width/2 × height` (world − `Origin.position`)
through `rider.playerCamera.WorldToScreenPoint`; bail if any corner `z <= 0`;
GUI y = `Screen.height - screenPoint.y`. Take min/max for the rect. GTA feel:

- While acquiring: white brackets, oversized ×1.6 shrinking to ×1.0 with lock
  progress, alpha blinking with the same period as the beep loop.
- Locked: solid red with a fast subtle pulse.
- Draw as 4 corner L-brackets + a 25 %-alpha 1 px full outline (1×1 solid
  Texture2D cache pattern from [IMGUI Tracker Stack](IMGUI%20Tracker%20Stack.md)).

## Synthesized lock-on audio (no ripped assets)

Both states are looping `AudioClip.Create` clips on one 2D AudioSource
(`spatialBlend 0`); switching the clip on state change restarts the loop:

- **Acquiring beeps**: one clip = short blip + silence (e.g. 65 ms @ 1170 Hz
  sine with 4 ms attack / 14 ms release envelope, in a 0.32 s buffer), looped —
  **beep cadence = clip length**, no timers.
- **Locked tone**: continuous sine loop that's seamless because the loop length
  holds an **exact integer number of cycles** (1560 Hz × 0.25 s = 390 cycles;
  add a 2nd harmonic at half amplitude-ish for an "avionics" timbre — also
  integer cycles). No crossfade needed.

Same `AudioClip.Create`/`SetData` approach as the Oppressor's jet loop and
[Synthesized Drone Rotor Audio](Synthesized%20Drone%20Rotor%20Audio.md).
