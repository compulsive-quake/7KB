# Entity Damage, Body Parts & Dismemberment

Part of the [7DTD Modding Knowledgebase](README.md). How to damage a **specific
body part** of an entity from mod code, make limbs actually come off, and control
the blood/sound feedback that rides along. Written while building the Shredder
trap's "hundreds of blades" grinding (`Shredder/src/ShredderBladeStorm.cs`,
2026-07). See also [Blocks](Blocks.md) and [Entities](Entities.md).

## The one call that does everything: `ItemActionAttack.Hit`

`ItemActionAttack.Hit(WorldRayHitInfo hitInfo, int attackerEntityId, …)` is the
same entry point melee weapons, bullets and the vanilla blade trap use. Feed it a
hand-built `WorldRayHitInfo` and you get damage, dismemberment, blood particles,
impact sounds, pain reactions, kill XP and net sync for free. Vanilla's own
example is `SpinningBladeTrapBladeController.Update` (raycasts from the blade hub
and passes `Voxel.voxelRayHitInfo.Clone()` straight through).

Which branch it takes is decided **entirely by `hitInfo.tag`**:

- `tag.StartsWith("E_")` → entity damage branch.
- `tag.StartsWith("E_BP_")` → additionally resolves a body part, and finds the
  entity via `RootTransformRefEntity.FindEntityUpwards(hitInfo.transform)`.
- otherwise → block/terrain branch.

## Hitting a chosen body part without a raycast

You do **not** need a physics raycast. Point the hit info at the bone yourself:

```csharp
Transform bone = entity.emodel.GetHitTransform(BodyPrimaryHit.LeftLowerLeg);
// public virtual Transform GetHitTransform(BodyPrimaryHit) on EModelBase;
// returns physicsBody.GetTransformForColliderTag("E_BP_LLowerLeg") — null if the
// entity class has no PhysicsBody layout.
hit.tag       = bone.tag;                          // "E_BP_LLowerLeg"
hit.transform = bone;
hit.hit.pos   = bone.position + Origin.position;   // scene → logical world
hit.fmcHit.pos = hit.hit.pos;                      // ← see below, easy to miss
```

**How the engine reads the part back** (this is why both the name and the tag
matter): `Hit` stores `hitInfo.transform.name` into the `DamageSourceEntity`, and
`DamageSource.GetEntityDamageBodyPart` later does
`emodel.GetHitTransform(damageSource)` → `bipedRootTransform.FindInChilds(name)` →
`TagToBodyPart(thatTransform.tag)`. So it round-trips **by bone name** and then
reads the **Unity tag** off whatever it found. Pass a transform that is both
findable under `bipedRootTransform` and carries its `E_BP_*` tag (a bone from
`GetHitTransform` satisfies both). If the tag is missing the hit still damages,
it just resolves to `EnumBodyPartHit.None` — no dismember, no headshot bonus.

Tag ↔ enum map (`DamageSource.TagToBodyPart`): `E_BP_Head`, `E_BP_Body`,
`E_BP_LArm`/`E_BP_LLowerArm`, `E_BP_RArm`/`E_BP_RLowerArm`, `E_BP_LLeg`/
`E_BP_LLowerLeg`, `E_BP_RLeg`/`E_BP_RLowerLeg`, `E_BP_Special`. The collider
layout itself comes from `Data/Config/physicsbodies.xml` (tag + bone path per
collider); `PhysicsBodyInstance.bindCollider` adds a `RootTransformRefEntity` to
each bound bone, which is what makes `FindEntityUpwards` work.

### Which limbs are still attached: `EntityAlive.bodyDamage.Flags`

`BodyDamage.Flags` is a bitmask of *missing* parts — `cNoHead = 1`,
`cNoArmLUpper = 2`, `cNoArmLLower = 4`, `cNoArmRUpper = 8`, `cNoArmRLower = 16`,
`cNoLegLUpper = 32`, `cNoLegLLower = 64`, `cNoLegRUpper = 128`,
`cNoLegRLower = 256` (`cCrippledLegL/R = 4096/8192`). Poll it to skip targeting
parts that are already gone, and diff it between ticks to detect "a limb just came
off" for your own gore sfx.

## `fmcHit`, not `hit`, positions the impact particle

`WorldRayHitInfo` carries **two** `HitInfoDetails`: `hit` and `fmcHit`. Damage
uses `hit.pos`, but the impact-particle spawn at the end of `Hit` uses
`hitInfo.fmcHit.pos` (and `fmcHit.blockFace` for its rotation). A hand-built
hit info that only fills `hit` sprays every blood puff at **world origin** —
silently, with no error. Fill both. `BlockFace.Top` (= 0, the default) maps to
`Quaternion.identity`, which is the "spray upward" orientation.

## Dismemberment: the chance is multiplied by the damage fraction

`EntityAlive.GetDismemberChance`:

```
chance = (source.DismemberChance < 100)
       ? source.DismemberChance * damageFraction * EntityClass.DismemberMultiplier{Head|Arms|Legs}
       : 100                                   // >= 100 forces a guaranteed dismember
then: dismember if rand.RandomFloat <= chance
```

`damageFraction` is that hit's damage over max health, so **the same
`_dismemberChance` value behaves completely differently depending on hit size**.
A weapon landing 30 % of a zombie's health passes ~0.05–1.5; a trap landing 2 % of
its health per tick needs ~1.0–3.0 to feel the same. Class multipliers are 1.0
for normal zombies, 0.7 feral, 0.4 radiated.

Consequences worth knowing:

- **A stunned/ragdolled victim cannot lose legs.** `CheckDismember` early-returns
  when the part is a leg and `bodyDamage.CurrentStun != None || sleepingOrWakingUp`.
  If you want progressive dismemberment, do **not** knock the victim down or call
  `emodel.DoRagdoll` while it is alive — keep it on its feet and let leg loss turn
  it into a crawler (`_dmResponse.TurnIntoCrawler`) naturally.
- **Head dismember on a live entity is an instant kill** — the engine sets
  `_dmResponse.Strength = Health`.
- On a **fatal** hit the engine floors the dismember chance at 0.3 (head) / 0.5
  (body), so killing blows usually take something off.

## Controlling the blood and the noise

`Hit` derives both the particle and the sound from
`_attackingDeviceMadeOf` + the target's surface category
(`EntityClass` property `SurfaceCategory`, `"organic"` for zombies):

- particle `impact_{madeOf}_on_{surface}` → e.g. `impact_metal_on_organic`
- sound `{madeOf}hit{surface}` → e.g. `metalhitorganic`

Three levers, useful when you are landing many hits per second:

| Want | Do |
|---|---|
| No particle **and** no sound (cheap hit) | set bit `8` in `_flags` (`_flags = 3 \| 8`) |
| Particle but **silent** | `_hitSoundOverrides[surface] = ""` — empty name skips `Manager.Play` |
| A different impact sound | `_hitSoundOverrides["organic"] = "chainsawhitorganic"` |

`_flags` bits otherwise: `1` = refresh attacker toolbelt, `2` = credit the kill as
an **electrical trap** kill (trap XP path), `4` = harvesting. Vanilla traps pass
`3`.

**Gotcha:** the override lookup is `_hitSoundOverrides.ContainsKey(text2)` with no
null guard, so an entity class without a `SurfaceCategory` throws
`ArgumentNullException` *after* the damage has been applied. Wrap the call.

Handy stock sounds for gore work (from `Data/Config/sounds.xml`):
`chainsawhitorganic` (saw eating meat, 6 clips, MaxVoices 5), `gore_head`,
`gore_limb_3s`, `gore_limb_6s`. Play world sounds with
`Audio.Manager.BroadcastPlay(Vector3 logicalWorldPos, string soundName)`.

### Particle spawn cap

`ParticleEffect.SpawnParticleEffect` keeps at most **3 concurrent instances of the
same particle per causing entity** (`cEntitySameParticleMax`) unless
`_forceCreation: true`. For deliberate bursts call
`GameManager.SpawnParticleEffectServer(pe, -1, _forceCreation: true, _worldSpawn: true)`
and throttle the rate yourself — positions passed with `_worldSpawn: true` are
logical world coords (the spawner subtracts `Origin.position`).
Probe names with `ParticleEffect.IsAvailable(name)` rather than hard-coding: the
`particleeffects` addressables load during world load, so an early probe can fail
— keep re-probing until one resolves.

## Pattern: "hundreds of blades" — split one tick into many cuts

To make a grinder/saw read as *shredding* rather than *clubbing*, keep the DPS
identical but deliver it as a burst of small hits per tick, each aimed at a
different bone, biased toward the parts lowest in the machine:

- 4 cuts per victim per 0.05 s tick ≈ 80 wounds/sec, each with its own dismember
  roll — limbs come off progressively and the body is eaten from the bottom up.
- Per-cut damage still floors at 1 (`Utils.FastMax(1, …)` inside `Hit`), so carry
  the fractional remainder between ticks instead of rounding each cut.
- Throttle feedback on a **timer**, not per cut (blood ≤ 20/s, impact sound ≤ 8/s),
  so the rate doesn't scale with how many bodies are in the hopper.
- Budget the total cuts per tick across all victims — `Hit` is not free.

Reference implementation: `Shredder/src/ShredderBladeStorm.cs`.

## How this was verified

`ilspycmd -t <Type> "<game>/7DaysToDie_Data/Managed/Assembly-CSharp.dll"` on
`ItemActionAttack`, `EntityAlive`, `DamageSource`, `EModelBase`,
`PhysicsBodyInstance`, `ParticleEffect`, `SpinningBladeTrapBladeController`.
