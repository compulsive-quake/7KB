# Runtime Procedural VFX (no asset bundle)

Part of the [7DTD Modding Knowledgebase](README.md). Covers VFX built at
runtime in C# from raw Unity primitives (LineRenderer, ParticleSystem, Light)
plus procedurally-generated textures/materials — i.e. effects that ship **no
`.unity3d` bundle**. For bundle-based assets see [Asset Bundles](Asset%20Bundles.md).

---

## The big gotcha: `HideAndDontSave` on procedural textures & materials

A procedural `Texture2D`/`Material` you build at runtime and hold in a `static`
field **will be destroyed out from under you** by the engine. 7DTD calls
`Resources.UnloadUnusedAssets()` on world transitions, save load, chunk
streaming, etc. That pass destroys any asset Unity thinks is unreferenced — and
a plain C# `static` reference does **not** count as a reference it respects.

Symptom: the first spawn of an effect looks right; later spawns render with a
**fallback texture — most commonly the most-recently-bound UI font atlas**, so
"all my projectiles/particles suddenly look like glyphs." The C# wrapper is
still non-null, but its underlying GPU texture/shader was freed.

Fix: set `hideFlags = HideFlags.HideAndDontSave` on the cached texture/material
so the cleanup pass skips it.

```csharp
var tex = new Texture2D(64, 64, TextureFormat.RGBA32, false);
tex.hideFlags = HideFlags.HideAndDontSave;   // survives UnloadUnusedAssets
// ... fill pixels, tex.Apply();

var baseMat = new Material(mat);
baseMat.hideFlags = HideFlags.HideAndDontSave;
```

Also guard the cache against partial-destroy states — check the shader too, not
just the material, before reusing: `if (cached != null && cached.shader != null)`.

## Particle material from a known additive shader (don't borrow scene materials)

To get an additive particle material, build from a named shader rather than
copying whatever material happened to be loaded (a borrowed scene torch/fire
material can have its texture unloaded later, giving the ghost-texture bug
above). Probe a fallback chain and clone:

```csharp
string[] shaderNames = {
    "Legacy Shaders/Particles/Additive", "Particles/Additive",
    "Mobile/Particles/Additive", "Particles/Standard Unlit",
    "Unlit/Transparent", "Sprites/Default" };
Material mat = null;
foreach (var n in shaderNames) { var s = Shader.Find(n); if (s != null) { mat = new Material(s); break; } }
mat.mainTexture = softCircleTexture;
mat.color = color;
if (mat.HasProperty("_TintColor")) mat.SetColor("_TintColor", color);
```

A 64×64 soft-circle (radial alpha falloff, squared) makes particles read as
fuzzy round glows instead of hard squares.

## Procedural lightning (LineRenderer)

A convincing bolt = fractal **midpoint displacement** for the trunk path
(subdivide each segment, perturb the midpoint perpendicular to the axis, halve
the amplitude each generation) + a few deflected sub-branches + per-frame
**visibility flicker** + a short `lifetime` (~0.4s) so it strobes like a real
strike rather than a steady beam. Scale the displacement to bolt length
(`max(floor, length * ratio)`) or long bolts look near-straight. Add a point
`Light` at the impact end, faded with the bolt, for the flash. `LineRenderer`
+ `Light` only — both in `UnityEngine.CoreModule` (no ParticleSystem module
reference needed). Reference implementation: zPhone `src/apps/God/GodLightning.cs`
(`GodLightningBolt`), ported from the Scepters mod.

## Beam LineRenderer flares wide where it meets a surface (depth test)

A `LineRenderer` "laser" drawn from a muzzle to a raycast hit point, using a
**depth-tested** shader (`Sprites/Default`, or any default sprite/unlit shader),
flares into a wide blob at its endpoint whenever the beam runs nearly parallel
to — or grazes at a shallow angle — the surface it lands on. The thin end-quad
sits coplanar with the block face and z-fights / partially clips against it, so
the flat ribbon smears across the surface. Symptom report: "the laser gets wide
on certain blocks" — i.e. only at certain look angles, not a constant width.

Fix: draw the beam with **depth testing off** so it never interacts with the
surface — it renders as a clean constant-width line on top. The muzzle→crosshair
path is always clear line-of-sight (it *is* the targeting ray), so drawing over
the world is correct. `Sprites/Default` has no settable `ZTest`, so switch to
`Hidden/Internal-Colored`, which exposes `_ZTest`/`_ZWrite`/`_SrcBlend`/
`_DstBlend`/`_Cull` as material ints and honours the LineRenderer's per-vertex
colours:

```csharp
Shader s = Shader.Find("Hidden/Internal-Colored");   // always present in builds
Material mat = new Material(s) { color = Color.white };
mat.SetInt("_ZTest", (int)UnityEngine.Rendering.CompareFunction.Always);
mat.SetInt("_ZWrite", 0);
mat.SetInt("_SrcBlend", (int)UnityEngine.Rendering.BlendMode.SrcAlpha);
mat.SetInt("_DstBlend", (int)UnityEngine.Rendering.BlendMode.OneMinusSrcAlpha);
mat.SetInt("_Cull", (int)UnityEngine.Rendering.CullMode.Off);
mat.renderQueue = 4000;                               // after transparent geometry
```

Keep a `Sprites/Default` fallback if `Shader.Find` returns null. Reference:
Airstrike `src/ItemActionAirstrikeDesignator.cs` (`CreateLaserMaterial`).

## Beam LineRenderer balloons wide when aimed toward/away from the camera (alignment)

Separate from the depth-test flare above. A `LineRenderer` left on its default
`LineAlignment.View` (camera-facing billboard) balloons into a wide flat ribbon
whenever a segment runs nearly **parallel to the camera's view ray** — the
camera-facing quad becomes ill-conditioned and the width compensation
overshoots. Symptom: "the laser stretches/gets wide at certain angles" even
though the configured width is millimetres. This persists *after* the depth-test
fix because it's a geometry/alignment issue, not a surface interaction.

Fix: switch to `LineAlignment.TransformZ` so width is laid in a stable plane
whose normal is `transform.forward`, then each frame aim that forward at the
camera **while keeping it perpendicular to the beam**. The naive
`LookRotation(-cam.forward, cam.up)` does NOT work: when you aim down the beam's
length (e.g. looking down at a near-ground target) the beam runs (anti)parallel
to `cam.forward`, so the normal lines up with the beam axis, the width cross
product `cross(beamDir, normal)` collapses to zero, and it balloons again — same
bug, worst spot. Instead project the beam direction out of the camera→start
vector and use the leftover in-plane component, which is perpendicular to the
beam by construction:

```csharp
Vector3 beamDir   = (targetWorld - startWorld).normalized;
Vector3 toCam     = camWorld - startWorld;
Vector3 normal    = toCam - Vector3.Dot(toCam, beamDir) * beamDir; // strip beam axis
if (normal.sqrMagnitude < 1e-6f) normal = cam.up;                  // aimed at the eye
transform.rotation = Quaternion.LookRotation(normal.normalized, beamDir);
```

The beam then renders a constant-width line across all look angles and only
thins (realistically) when aimed straight at the viewer — a flat ribbon
containing a view-parallel beam *cannot* also face the viewer broadside, so
thinning is the correct result, not ballooning. Reference: Airstrike
`src/ItemActionAirstrikeDesignator.cs` (`EnsureLaser` sets the alignment;
`AirstrikeLaserView.LateUpdate` builds the perpendicular normal).

Surface-hit data, if you ever need the normal instead: 7DTD's voxel raycast
returns `Voxel.voxelRayHitInfo` (`WorldRayHitInfo`); `.hit` is a `HitInfoDetails`
struct with `Vector3 pos`, `Vector3i blockPos`, `BlockFace blockFace`,
`distanceSq` — no `Vector3 normal` field, so derive a normal from `blockFace`
(Top=+Y, Bottom=−Y, North=+Z, South=−Z, East=+X, West=−X; Middle/None = unknown).

## Reusing a vanilla explosion VFX/SFX without doing damage

You don't have to hand-roll an explosion — the game's explosion prefabs are
loadable at runtime and ship the flames, smoke, scorch decal, dynamic light and
the boom AudioSource all in one. `WorldStaticData.prefabExplosions` is a
`Transform[100]` indexed by the `Explosion.ParticleIndex` items use (e.g. **13 =
thrown-grenade explosion**). They're loaded from Addressables
(`Prefabs/prefabExplosion{i}.prefab`) during world load, so the array may still
be `null`/empty before a world is up — guard for it.

`GameManager.ExplosionClient(...)` is what spawns one for a real explosion, but
it *also* runs `ApplyExplosionForce.Explode` and `QuestEventManager.DetectedExplosion`.
For a **purely cosmetic** blast (you apply your own entity damage/knockback, or
want zero gameplay effect), skip it and instantiate the prefab yourself:

```csharp
Transform[] fx = WorldStaticData.prefabExplosions;
if (fx != null && idx > 0 && idx < fx.Length && fx[idx] != null) {
    GameObject go = Object.Instantiate(fx[idx].gameObject, worldPos - Origin.position, Quaternion.identity);
    // the boom rides the prefab as a PlayOnAwake AudioSource — instantiating plays it.
    Object.Destroy(go, 5f);   // flames last ~1-2s; rest is idle tail
}
```

The explosion **sound is not played in code** — the explode path (`ItemClassTimeBomb`
→ `ExplosionServer` → `ExplosionClient`) never calls `Audio.Manager`; the boom is
an AudioSource on the prefab itself. So instantiating the prefab gives flames **and**
sound for free. (If you ever hit a silent instance, `GetComponentsInChildren<AudioSource>`
and `.Play()` the non-playing ones.)

**Perf gotcha #1 — strip the dynamic Light when spawning many at once.** Each prefab
carries a real-time point `Light`. A handful is fine; dozens spawned within a
fraction of a second (e.g. an airstrike landing ~24 impacts in ~0.4s) tanks FPS —
simultaneous real-time lights are a top cost. Strip them and the flames still read:

```csharp
foreach (var l in go.GetComponentsInChildren<Light>(true)) Object.Destroy(l);
```

**Perf gotcha #2 — the smoke/dust particle systems are overdraw bombs.** With the
lights gone, the next-biggest cost is fill-rate from the big semi-transparent
smoke/dust/dirt clouds. They're large quads that overlap heavily when many blasts
stack, blowing out overdraw. You can keep *only the flames* by destroying the
smoke/dust child systems by name and leaving the fire/spark/glow ones:

```csharp
string[] cut = { "smoke", "dust", "dirt", "debris", "twirl", "ground" };
foreach (var ps in go.GetComponentsInChildren<ParticleSystem>(true)) {
    var n = ps.gameObject.name.ToLowerInvariant();
    if (System.Array.Exists(cut, k => n.Contains(k))) Object.Destroy(ps.gameObject);
}
```

### Anatomy of `prefabExplosion13` (thrown-grenade), so you know what you're cutting

Root `prefabExplosion13` has **no AudioSource** — two MonoBehaviours instead:
`AudioPlayer { soundName="explosion_grenade", playOnDemand=0 }` (plays the boom on
spawn via the sound system, not an AudioSource — so don't go hunting for one) and
`TemporaryObject { life≈4.1 }` (self-destructs, so an explicit `Destroy(go, t)` is
only a safety net). Under child `v021`, the particle systems are:

| Keep (flames/sparks/glow — cheap) | Cut (smoke/dust/debris — overdraw) |
| --- | --- |
| `coreBlast_mainBlast` (fireball) | `coreBlast_mainBlastSmokeFill` |
| `coreBlast_centerBlast` | `coreBlast_outwardDust_blast` |
| `coreBlast_muzzleBlast` | `coreBlast_outwardDust_secondaryBlast` |
| `sparksSource` / `part_Sparks_A_pS` | `coreBlast_dirt` |
| `p_light` (additive glow card) | `coreBlast_twirl_A` / `coreBlast_twirl_B` |
| | `grenadeDebris` |
| | `groundDetector_card_C` / `groundBlast_C` |

The `coreBlast_*` / `*Smoke*` / `*Dust*` naming is a shared authoring template across
the explosion prefabs, so the keyword filter generalises to other indices.

**Making a flash linger.** The flame systems emit a single **burst at t=0**, so what
you see is bounded by each particle's `startLifetime` (~0.7s for the grenade's core
flame), *not* the system's `duration` (which is 3s but emits nothing after the burst).
To stretch the visible fire you must touch lifetime, not duration.

Two levers, with a sharp gotcha on the first:

- `TemporaryObject.SetLife(float seconds)` (public, on the explosion's root) — the
  game's own rescale: multiplies each qualifying child's `duration`/`startDelay`/
  `startLifetime` and sets the object's `life`. **Gotcha:** it derives ONE global
  factor `num3 = (life-1) / maxChildDuration`, and it skips systems whose duration
  `< maxChildDuration*0.5`. On `prefabExplosion13` a spark system has `duration=5`,
  so `maxChildDuration=5` and `SetLife(8)` yields only `7/5 = 1.4×` — the flames
  barely change. So SetLife is unreliable whenever child durations vary.
- `ps.main.simulationSpeed *= 0.33f` on each kept system — slows every system by the
  same factor (a particle living 0.7s now shows for ~2.1s), preserving the authored
  look. Side effect: the burst *expands* in slow motion too (mild bullet-time), which
  for fire reads fine. simulationSpeed does **not** extend the GO's life, so also widen
  it: set `temp.life = N` directly (its destroy coroutine reads `life` from `Start()`,
  which runs after your post-Instantiate code, so the field is picked up — no
  `Restart()` needed) and keep a backstop `Object.Destroy(go, N + margin)`.

For Airstrike, simulationSpeed + `temp.life` won out over SetLife for exactly the
varying-duration reason above.

**Inspecting a prefab offline (UnityPy).** The prefabs live in
`Data/Addressables/Standalone/prefabs_assets_all.bundle` (GameObject
`prefabExplosion{i}`); explosion **sounds** are in `automatic_assets_sounds/explosions.bundle`.
`UnityPy.load(bundle)` → walk a GameObject's `m_Component` (deref PPtr `m_PathID`
against an id→obj map) and `Transform.m_Children` to print the hierarchy; read a
MonoBehaviour's `m_Script.m_ClassName` and `read_typetree()` for its fields.

Reference: Airstrike `src/AirstrikeExplosionFx.cs` (lethal cannon impacts spawn the
grenade explosion, light-stripped + smoke/dust-pruned; entity damage applied in code).

## World → scene space

Effect positions must be converted from logical world coords to scene space by
subtracting `Origin.position` (the floating-origin offset) before you place a
GameObject: `go.transform.position = worldPos - Origin.position;`. Entity
`.position` is already logical-world, so a bolt between a cloud and an entity
passes `cloudPos - Origin.position` and `entity.position - Origin.position`.

## Porting a reflected cross-mod effect to native (drop the dependency)

zPhone's God "Kill Everything" originally reflected into
`Scepters.ScepterControlAPI.KillAllZombies` and refused to run when Scepters
wasn't installed. Because the bolt + particle-material code is pure Unity (no
Scepters types), it was copied verbatim into zPhone (`GodLightning.cs`,
renaming `LightningBolt` → `GodLightningBolt` to avoid cross-assembly
type-name ambiguity when both mods load). The only behaviour dropped was the
"skip Scepters-enchanted pets" check, which is meaningless without Scepters.
Lesson: if a reflected cross-mod feature has no hard dependency on the other
mod's *types*, porting the self-contained part is cleaner than guarding every
call site on `ModBridge.IsInstalled(...)`.
