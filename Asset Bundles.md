# Asset Bundles

Part of the [7DTD Modding Knowledgebase](README.md). Covers loading custom 3D models, particles, sounds, and other assets from Unity asset bundles.

---

## Overview

7DTD mods use Unity `.unity3d` asset bundles to load custom 3D models, prefabs, audio clips, particle effects, and other Unity assets. Bundles go in the mod's `Resources/` folder.

---

## Referencing Bundles in XML

### Block models

Use the `#@modfolder:` prefix to reference models in asset bundles:

```xml
<property name="Model" value="#@modfolder:Resources/mymodel.unity3d?myprefab.prefab"/>
```

Format: `#@modfolder:Resources/<bundle>?<asset path inside bundle>`

### Shape requirement

Blocks using custom models must set:
```xml
<property name="Shape" value="ModelEntity"/>
```

---

## Loading Vanilla 7DTD Assets from C#

Vanilla prefabs, materials, particles, etc. live inside the game's main asset
bundle (`data.unity3d`). The game exposes a single resolver — `DataLoader` —
that understands the `@:` prefix as "look in the vanilla bundle":

```csharp
GameObject rocket = DataLoader.LoadAsset<GameObject>(
    "@:Other/Items/Weapons/Ranged/RocketLauncher/rocketPrefab.prefab");

Transform entityPrefab = DataLoader.LoadAsset<Transform>(
    "@:Entities/Player/Common/AnimControllers/...");
```

This is how you reuse vanilla visuals/sounds without shipping copies. Concrete
recipe used by RocketTurret (rocket projectile visual): instantiate
`rocketPrefab`, then strip the components that would drive its own behavior
(Rigidbody, colliders, Projectile/Rocket MonoBehaviours) so you keep the mesh,
trail ParticleSystems, and AudioSource while flying it manually. Cache the
loaded prefab in a static — `LoadAsset` is not free per call.

> **Vanilla projectile prefabs ship with effect children INACTIVE.**
> `rocketPrefab` has Smoke/flame/glow children with `activeSelf=false`; the
> vanilla `Launcher` action calls `OnActivateItemGameObjectReference.ActivateItem(true)`
> at fire time to wake them. If you instantiate the prefab yourself, you'll get
> just the bare mesh — no smoke trail, no flame. Fix: recursively walk children
> after `Instantiate` and `SetActive(true)` on each, then `Play()` any
> `ParticleSystem` that isn't already playing (instantiating a deactivated
> hierarchy can skip the initial PlayOnAwake pulse). See `RocketTurret`'s
> `ActivateTrailEffects` in `RocketTurretLaunchManager.cs`.

For the firing/weapon sound, use the sound *name* from `sounds.xml` instead of
loading the clip directly:

```csharp
using Audio;

// Networked: plays locally AND broadcasts to other clients in earshot.
GameManager.Instance.PlaySoundAtPositionServer(
    worldPos, "m136_fire", AudioRolloffMode.Logarithmic, 80);

// Local-only fallback if no GameManager (e.g. very early in load):
Manager.Play(worldPos, "m136_fire");
```

`Manager` lives in the `Audio` namespace. Common useful vanilla sound IDs:
`m136_fire` (rocket launcher), `m136_reload_*`, weapon-class fires, etc. —
grep `Data/Config/sounds.xml` for `SoundDataNode name=`.

The C# project needs references to `UnityEngine.ParticleSystemModule.dll` and
`Unity.Addressables.dll` (both ship in `7DaysToDie_Data/Managed/`) to use
`ParticleSystem` APIs and `DataLoader.LoadAsset` respectively.

---

## Loading Bundles from C#

### From mod Resources folder

```csharp
string modPath = mod.Path; // from IModApi.InitMod
string bundlePath = Path.Combine(modPath, "Resources", "myassets.unity3d");
AssetBundle bundle = AssetBundle.LoadFromFile(bundlePath);
```

### Loading specific assets

```csharp
// Load a prefab
GameObject prefab = bundle.LoadAsset<GameObject>("myprefab.prefab");

// Load with case-insensitive fallback
GameObject TryLoad(AssetBundle b, string[] candidates)
{
    string[] allNames = b.GetAllAssetNames();
    foreach (string name in candidates)
    {
        var go = b.LoadAsset<GameObject>(name);
        if (go != null) return go;

        // Try case-insensitive match
        string lower = name.ToLowerInvariant();
        foreach (string asset in allNames)
        {
            if (asset.ToLowerInvariant().EndsWith(lower))
            {
                go = b.LoadAsset<GameObject>(asset);
                if (go != null) return go;
            }
        }
    }
    return null;
}
```

### Checking already-loaded bundles

Unity may have already loaded the bundle (e.g., from the block model reference). Check first:

```csharp
foreach (AssetBundle b in AssetBundle.GetAllLoadedAssetBundles())
{
    if (b.name.Contains("myassets"))
    {
        // Use this bundle
        break;
    }
}
```

> **Important:** `AssetBundle.LoadFromFile` will fail if the bundle is already loaded. Always check `GetAllLoadedAssetBundles()` first.

---

## Instantiating Prefabs at Runtime

Attach to a block's transform via `BlockEntityData`:

```csharp
// Get the block's entity data
BlockEntityData bed = chunkSync.GetBlockEntity(blockPos);
if (bed != null && bed.bHasTransform)
{
    Transform parent = bed.transform;

    // Find a named socket in the hierarchy
    Transform socket = FindDeepChild(parent, "MySocket");
    if (socket != null)
        parent = socket;

    GameObject instance = Object.Instantiate(prefab, parent, false);
    instance.transform.localPosition = Vector3.zero;
    instance.transform.localRotation = Quaternion.identity;
}
```

### Finding child transforms recursively

The game's transforms can be deeply nested. Use BFS:

```csharp
static Transform FindDeepChild(Transform root, string name)
{
    Queue<Transform> queue = new Queue<Transform>();
    queue.Enqueue(root);
    while (queue.Count > 0)
    {
        Transform t = queue.Dequeue();
        if (t.name == name) return t;
        for (int i = 0; i < t.childCount; i++)
            queue.Enqueue(t.GetChild(i));
    }
    return null;
}
```

---

## Playing a loose .wav at runtime (no bundle)

You don't need an asset bundle (or `Config/sounds.xml`) to play a custom sound —
ship a `.wav` with the mod and build an `AudioClip` from it on the fly. This
keeps audio out of the Unity bake pipeline entirely.

- **Reference only `UnityEngine.AudioModule`** (for `AudioClip`/`AudioSource`).
  Do **not** pull in UnityWebRequest just to load audio — parse the WAV yourself
  and `AudioClip.Create(name, perChannelSamples, channels, sampleRate, false)` +
  `clip.SetData(interleavedFloats, 0)`. Synchronous, main-thread, no async.
- **WAV parse:** validate `RIFF`/`WAVE`, then walk chunks from offset 12
  (`id`=4 bytes, `size`=4 bytes LE) looking for `fmt ` (format/channels/rate/bits)
  and `data`. Convert 16-bit PCM via `BitConverter.ToInt16(...) / 32768f`. Chunks
  are word-aligned (pad odd sizes by 1). Don't assume data starts at byte 44.
- **Deploy gotcha:** `deploy.ps1` `/MIR`-excludes the lowercase `resources/` dir
  *and wipes the destination every deploy*, so loose source files there never
  ship. Copy the specific `.wav`s into a non-excluded dest folder (e.g.
  `<dest>/Sounds`) in the post-`/MIR` block, exactly like the unity bundle is
  copied. Load them at runtime from `Path.Combine(AirstrikeModApi.ModPath, "Sounds", name)`.
- **The sharper version of that gotcha (Supra, 2026-07):** on Windows,
  `resources/` (model sources) and `Resources/` (shipping bundle) are the SAME
  directory — NTFS and robocopy `/XD` are case-insensitive. A Unity build that
  writes `Resources/foo.unity3d` while scaffold sources sit in `resources/`
  silently merges them, and excluding `resources` from deploy then excludes the
  bundle too. Keep model/source assets in a differently-named folder
  (`model-src/`) and let `Resources/` ship normally.
- **Play 3D:** `go.AddComponent<AudioSource>()` with `spatialBlend=1`,
  `rolloffMode=Linear`, `min/maxDistance`, `dopplerLevel=0` (keep authored pitch),
  then `Destroy(go, clip.length + 0.25f)`. Parent the source to a moving object
  (e.g. the jet) to make the sound travel with it. Position is origin-relative:
  set `transform.position = worldPos - Origin.position`.

See `Airstrike/src/AirstrikeAudio.cs` for the full implementation (engine + cannon
flyby sounds choreographed to the strafe — added June 2026).

---

## Procedural runtime VFX (no baked assets): muzzle flash example

You can build glow/flash effects entirely in code, no bundle assets:

- **Glow sprite without a texture asset:** generate a `Texture2D` radial blob
  (white-hot core → transparent black edge, `a*a` for a tight core) and put it on
  a `GameObject.CreatePrimitive(PrimitiveType.Quad)` (delete its `MeshCollider`).
- **Shader, safely:** don't assume a specific transparent shader exists — try a
  chain and take the first `Shader.Find` hit: `Legacy Shaders/Particles/Additive`
  → `Particles/Additive` → `Mobile/Particles/Additive` → `Sprites/Default` →
  `Unlit/Transparent`. Additive (black = transparent, bright = adds to scene) gives
  the best glow with no card edges. Set `_TintColor` (particle shaders) **and**
  `_Color` (sprite shaders) — the irrelevant one is ignored. Degrade gracefully to
  a light-only flash if none resolve. (This is the same shader-availability caution
  as the magenta explosion-decal issue — see Items.md.)
- **Framerate-independent strobe:** drive on/off from `Time.time`, not frame count:
  `bool on = (Mathf.FloorToInt(Time.time * 2f * hz) & 1) == 0;` gives `hz` pulses/s
  at any FPS. (Above ~half the refresh rate it aliases into a rapid flicker, which
  reads fine for a muzzle flash. The GAU-8 is ~3900 rpm = **65 flashes/sec**.)
- **Billboard:** in `LateUpdate`, `quad.rotation = Quaternion.LookRotation(quad.position - Camera.main.transform.position)`.
  Both positions are floating-origin-relative, so no `Origin` correction needed
  between them.
- Add a strobing `Light` alongside the quad for environmental punch (works in
  daylight where an additive quad alone can wash out).

See `Airstrike/src/AirstrikeMuzzleFlash.cs` (added June 2026).

---

## Audio from Asset Bundles

Audio clips and `AudioSource` components can be embedded in prefabs. Find them by name:

```csharp
static AudioSource FindAudioSourceByName(Transform root, string name)
{
    foreach (Transform t in root.GetComponentsInChildren<Transform>(true))
    {
        if (t.name == name)
            return t.GetComponent<AudioSource>();
    }
    return null;
}
```

Control playback:
```csharp
AudioSource startSound = FindAudioSourceByName(blockTransform, "startSound");
AudioSource loopSound = FindAudioSourceByName(blockTransform, "loopSound");

// Start: play start sound, then delay loop
startSound.Play();
float startLen = startSound.clip?.length ?? 0f;
loopSound.PlayDelayed(startLen);
loopSound.loop = true;

// Stop: stop all, play stop sound
startSound.Stop();
loopSound.Stop();
stopSound.Play();
```

### Distance-based volume

Add a MonoBehaviour component for distance attenuation:

```csharp
public class DistanceAudio : MonoBehaviour
{
    public float MaxDistance = 35f;
    private AudioSource[] sources;
    private float[] baseVolumes;

    void LateUpdate()
    {
        var player = GameManager.Instance?.World?.GetPrimaryPlayer();
        if (player == null) return;

        float dist = Vector3.Distance(player.transform.position, transform.position);
        float volume = 1f - Mathf.Clamp01(dist / MaxDistance);

        for (int i = 0; i < sources.Length; i++)
            sources[i].volume = baseVolumes[i] * volume;
    }
}
```

---

## Particle Effects from Bundles

Load particle prefabs the same way as other assets. Control emission:

```csharp
ParticleSystem[] particles = go.GetComponentsInChildren<ParticleSystem>(true);

// Enable/disable emission
foreach (var ps in particles)
{
    var emission = ps.emission;
    emission.enabled = isRunning;
}

// Play/stop
foreach (var ps in particles)
{
    if (isRunning) ps.Play();
    else ps.Stop(true, ParticleSystemStopBehavior.StopEmitting);
}
```

---

## Unity Project Setup for 7DTD Prefabs

### Headless build wrapper

PowerShell wrappers that launch Unity batch mode should resolve the mod root from `MODFORGE_MOD_DIR` when present, otherwise from `$MyInvocation.MyCommand.Path`, and should read `UnityProject/ProjectSettings/ProjectVersion.txt` to find the intended Hub editor version. Accept both `UNITY_EXE` and `UNITY_EDITOR_PATH` as overrides.

Prefer launching `Unity.exe` with `Start-Process -Wait -PassThru` and writing logs to `UnityProject/Logs/headless.log` instead of relying on `& $UnityExe ... -logFile -`. Unity is a GUI-subsystem executable on Windows, so direct PowerShell invocation can return unreliable `$LASTEXITCODE` behavior. On failure, tail the local log.

Before invoking Unity, fail fast if `UnityProject/Temp/UnityLockfile` exists. Unity can appear to succeed while doing nothing when the project is already open in the editor. After invocation, verify the expected `Resources/*.unity3d` bundle exists before reporting success.

### Generated Unity projects

When scaffolding a Unity project outside the editor, include `ProjectSettings/ProjectVersion.txt`. Unity Hub uses this file to detect the intended editor version; without it, Hub shows a "Restricted Editor Version" warning and makes the user choose an installed editor manually. Match the version to the Unity editor used by the local workspace, for example:

```text
m_EditorVersion: 2022.3.62f3
m_EditorVersionWithRevision: 2022.3.62f3
```

### ModForge Sketchfab imports

ModForge's Sketchfab asset browser imports only downloadable Sketchfab models. Search is public, but archive import uses Sketchfab's authenticated `GET /v3/models/{uid}/download` endpoint; provide an OAuth access token in the UI. Imports create or reuse `<mod>/UnityProject`, unpack the glTF archive under `UnityProject/Assets/ModForge/Sketchfab/<asset-name>/`, add editor scripts for a headless Sketchfab AssetBundle build, and write `<mod>/build-sketchfab-assets.ps1`. Attribution is tracked at the mod root in `attribute.md` with a `sketchfab:<uid>` marker, model URL, author, and license.

### Replacing imported source models

Headless import scripts should mirror source folders, not only copy new files over old ones. When a modder replaces a model source folder, delete stale generated Unity assets (source copies, generated prefabs, and selected prefab aliases) before recreating them. Otherwise removed source models can remain in `Assets/` and continue to be bundled accidentally. Once a final model is chosen, prefer generating the stable XML-facing prefab directly instead of keeping extra candidate/alias prefabs in the bundle.

### Texture import timing

If setup scripts copy textures into `Assets/` and call `AssetDatabase.Refresh`, Unity imports those textures before later setup code can adjust their `TextureImporter`. Materials that bind a normal map during that first import can show "texture must be marked as a normal map" warnings even if the setup method fixes the importer afterward. Add an `AssetPostprocessor.OnPreprocessTexture` for generated asset folders so filenames containing `normal` are marked as `TextureImporterType.NormalMap` during the initial import pass.

### TagManager.asset

The Unity project **must** use the 7DTD TagManager.asset, not Unity's default. Download it from the [7D2D/Templates-and-Utilities](https://github.com/7D2D/Templates-and-Utilities) repo (use the `A21TagManager.zip`) and replace `ProjectSettings/TagManager.asset`.

This defines all the custom tags and layers the game expects:

#### Required tags for block prefabs

| Tag | Purpose |
|-----|---------|
| `T_Block` | Block-type objects — required for the game's interaction raycast to recognise the block for E-press activation |
| `T_Mesh_B` | Mesh-type objects like plants |
| `T_Deco` | Decoration objects |

#### Prefab structure requirements

For `Shape="ModelEntity"` blocks:

1. **Root GameObject** — empty parent, receives the `T_Block` tag and a `BoxCollider` sized to cover the block. No mesh components.
2. **Child FBX objects** — nested under root, hold `MeshFilter`, `MeshRenderer`, and materials. No colliders on children.
3. The root `BoxCollider` is what the game's interaction raycast hits. Without `T_Block` on the root, the block will render in-game but **cannot be interacted with** (no E-press prompt).
4. Child `MeshCollider`s should be removed — they cause lag and can intercept the raycast before it reaches the root `BoxCollider`.

> **Without the 7DTD TagManager**, the `T_Block` tag won't exist in your Unity project and the prefab will export with `Untagged`, making the block non-interactive in-game (even though it works in the prefab editor, which has its own simplified interaction).

#### UnityYAML parse failure: empty layer entries need a trailing space

If Unity refuses to open the project with `Unable to parse file ProjectSettings/TagManager.asset: [Parser Failure at line N: Expect ':' between key and value within mapping]`, check the empty slots in the `layers:` list. Unity's own serializer writes empty entries as `  - ` (dash **plus trailing space**); a hand-edited or script-generated file with a bare `  -` is valid YAML (PyYAML parses it fine) but UnityYAML chokes on it and reports the error at an unrelated later line (typically inside `m_SortingLayers`). Fix: `sed -i 's/^  -$/  - /' ProjectSettings/TagManager.asset`. Beware editors/pre-commit hooks that strip trailing whitespace — they reintroduce the problem.

#### Tag ORDER matters, not just tag presence (index serialization)

Asset bundles serialize a GameObject's tag as an **index** into the authoring project's tag list (custom tags start at 20000), and the game resolves that index against **its own** TagManager at load. Merely adding `T_Block` to your project is NOT enough — if it isn't at the same position as in the game's list, the prefab silently resolves to a *different* tag in-game. Real case (RocketTurret, 2026-06): a project whose only custom tag was `T_Block` shipped a root tagged index 20000, which the game resolved as `Item` → block rendered fine but had no E-prompt and took no melee/bullet damage.

The game's custom tag order (V 2.6 b14), indices 20000+: `Item, T_Mesh, B_Mesh, T_Mesh_B, T_Block, SB_Prefabs, ...` — so `T_Block` is **20004**. Mirror the game's full tag AND layer lists in `ProjectSettings/TagManager.asset` (the Templates-and-Utilities repo's TagManager does this; a hand-rolled one must match order exactly).

To dump the game's authoritative lists (UnityPy):

```python
import UnityPy
env = UnityPy.load(r'D:\...\7DaysToDie_Data\data.unity3d')
for obj in env.objects:
    if obj.type.name == 'TagManager':
        d = obj.read_typetree()
        print(d['tags'])    # custom tags, index 20000+
        print(d['layers'])  # layers 0-31
```

The same technique on the mod's `.unity3d` (read `GameObject` objects, print `m_Tag`) shows what index actually shipped.

**Why the tag drives everything:** decompiled `Voxel.raycastNew` checks `hit.collider.transform.tag == "T_Block"`, then `GameUtils.FindMasterBlockForEntityModelBlock` → `RootTransformRefParent.FindRoot(hitTransform)` → `chunk.GetBlockEntity(transform)` (exact reference match against `BlockEntityData.transform`). Any other tag means the hit never maps to a block: no activation prompt, no block damage. Layers don't matter for this — the chunk re-layers model colliders itself via `Utils.SetColliderLayerRecursively` at spawn.

**Runtime safety net:** a custom block class can force the tag on every model spawn, which fixes already-built bundles without a Unity rebuild:

```csharp
public override void OnBlockEntityTransformAfterActivated(WorldBase _world,
    Vector3i _blockPos, int _cIdx, BlockValue _blockValue, BlockEntityData _ebcd)
{
    base.OnBlockEntityTransformAfterActivated(_world, _blockPos, _cIdx, _blockValue, _ebcd);
    if (_ebcd != null && _ebcd.bHasTransform && _ebcd.transform != null
        && !_ebcd.transform.CompareTag("T_Block"))
        _ebcd.transform.tag = "T_Block";
}
```

---

## Sizing imported models correctly (1 unit = 1 metre)

7DTD treats **1 Unity unit as 1 metre**. A motorcycle is ~2.4–2.6m on its longest axis, a player ~1.85m, a 4×4 truck ~4.5m. Get this wrong and your prefab spawns the size of a house (real example: an Oppressor MK II shipped at 200m because three independent corrections stacked on each other).

### Three things that decide imported size

1. **Source-file unit.** FBX from Blender exports in metres by default. DAE/COLLADA carries `<unit name="meter" meter="0.01"/>` (Sketchfab exports default to centimetres). Don't trust the source-file unit to mean what you think — measure the imported result.
2. **`ModelImporter.useFileScale` + `globalScale`.** With `useFileScale = true`, Unity honours the file's unit (works for clean Blender FBX). With `useFileScale = false`, Unity ignores the file's unit and multiplies by `globalScale`.
3. **Any `localScale` on the prefab wrapper.** This compounds with the importer scale and is the easiest place to introduce a sizing bug.

### Single-source-of-truth recipe

**Pick ONE of `useFileScale=true` OR `useFileScale=false` + a single `globalScale`. Never combine import-time scaling with a runtime fit pass — the errors compound.**

```csharp
static void ConfigureModelImporter(string daePath)
{
    var importer = AssetImporter.GetAtPath(daePath) as ModelImporter;
    importer.useFileScale = false;
    importer.globalScale  = 1.3f;  // tuned empirically — see assertion below
    importer.isReadable   = true;
    importer.SaveAndReimport();
}
```

The wrapper prefab stays at `localScale = (1,1,1)`. No runtime rescale.

### Assert, don't silently rescale

After import, **measure** the renderer bounds and warn if they're off — but never silently multiply onto `wrapper.localScale` based on `Renderer.bounds`. World-space bounds can be stale right after instantiation, and a wrong measurement cascades into a 10–100× sizing bug.

```csharp
const float TargetLongestAxisMeters = 2.6f; // motorcycle-sized

static void AssertScaleNearTarget(Transform wrapper)
{
    var renderers = wrapper.GetComponentsInChildren<Renderer>(true);
    if (renderers.Length == 0) return;

    Bounds b = renderers[0].bounds;
    for (int i = 1; i < renderers.Length; i++) b.Encapsulate(renderers[i].bounds);

    float longest = Mathf.Max(b.size.x, Mathf.Max(b.size.y, b.size.z));
    float ratio   = longest / TargetLongestAxisMeters;
    Debug.Log($"Imported longest-axis {longest:F3}m (target {TargetLongestAxisMeters}m, ratio {ratio:F2}×).");

    if (ratio < 0.8f || ratio > 1.2f)
    {
        Debug.LogWarning(
            $"Imported size {longest:F3}m is >20% off target {TargetLongestAxisMeters}m. " +
            $"Adjust ModelImporter.globalScale by {1f/ratio:F3}× and re-bake. " +
            "Do NOT add a runtime rescale.");
    }
}
```

### Tuning workflow

1. Bake the bundle once with `globalScale = 1`. Read the assertion log.
2. The log says e.g. `Imported longest-axis 200.000m (target 2.6m, ratio 76.92×)`. Multiply your current globalScale by `1/ratio`: `1 / 76.92 ≈ 0.013`. Set `globalScale = 0.013`. (Or if you started at 100, set it to `100 × 0.013 = 1.3`.)
3. Re-bake. Confirm the log now reads `ratio 1.00×`.
4. Open the Unity scene with the prefab. **Drop a 1.85m capsule next to it. Eyeball.** If you can't visually verify motorcycle-scale in the editor, no in-game test will save you.

### Optional: ship a runtime ModelScale knob for playtest

A `ModelScale` slider on the entity's tunable API (default 1.0, range 0.02–4.0) is cheap insurance for finding the *gameplay-correct* size after the bake is technically right. Once you find the value that feels right, **bake it back into `globalScale` and reset the slider default to 1.0** — the slider is a tuning knob, not a load-bearing fix.

---

## Gotchas

> **Missing T_Block tag** — If your ModelEntity block renders but can't be interacted with in-game, the prefab's root GameObject is likely missing the `T_Block` tag. Ensure the 7DTD TagManager.asset is installed and the root is tagged. Note that the tag is serialized **by index**, so the project's custom-tag *order* must match the game's (`T_Block` = 20004) — see "Tag ORDER matters" above.

> **Bundle already loaded** — If the game loaded the bundle via a block `Model` property, `AssetBundle.LoadFromFile` will fail. Use `GetAllLoadedAssetBundles()` first.

> **Asset paths are lowercase** — Unity normalizes asset bundle paths to lowercase. Use case-insensitive matching when searching for assets.

> **Dedicated server** — Don't load visual assets (particles, audio) on dedicated servers. Guard with `if (GameManager.IsDedicatedServer) return;`

> **Bundle file location** — The game resolves `#@modfolder:` to the mod's root directory. Place bundles in `Resources/` by convention.

> **Don't combine import-time scaling with a runtime fit pass** — measuring `Renderer.bounds` right after `PrefabUtility.InstantiatePrefab` and multiplying onto `wrapper.localScale` can produce wildly wrong sizes (a 2.6m bike became 200m in one real case). Set sizing via `ModelImporter.globalScale` at import time, then *assert* the result instead of silently fixing it.

> **Don't overwrite FBX-embedded materials** — If the FBX came with per-part colors/materials (common for multi-piece models like phones, weapons, vehicles), do **not** loop `renderer.sharedMaterial = someMat` in your setup script. That flattens the whole model to one material. Only replace materials when the FBX has none and you need to inject one (e.g. a single-material model like the NES console). To keep the FBX materials on the prefab and include them in the asset bundle, set on the `ModelImporter`:
> ```csharp
> importer.materialImportMode = ModelImporterMaterialImportMode.ImportStandard;
> importer.materialLocation   = ModelImporterMaterialLocation.InPrefab;
> importer.materialName       = ModelImporterMaterialName.BasedOnMaterialName;
> importer.materialSearch     = ModelImporterMaterialSearch.Local;
> ```
> `InPrefab` keeps the materials as sub-assets of the FBX, so they travel with the prefab into the bundle as dependencies. No separate material extraction needed.

---

## Emission map gotcha — whole model glows a flat color in-game

If a placed/held custom model glows a uniform bright color (e.g. neon green legs) that you can't reproduce in the Unity editor, suspect the Standard-shader emission slot. A model-setup script that does `mat.SetTexture("_EmissionMap", tex); mat.EnableKeyword("_EMISSION"); mat.SetColor("_EmissionColor", Color.white)` will light the model by that map at full intensity in-game — **even if the committed `.mat` YAML doesn't list `_EMISSION` in `m_ValidKeywords`**, because the bundle is rebuilt from the script, not the YAML. Placeholder emissive textures are frequently a solid color block (an artist's mask UV fill), so the whole model takes that color. Fix: stop applying the emission map and explicitly neutralize it — `DisableKeyword("_EMISSION")`, `SetColor("_EmissionColor", Color.black)`, `globalIlluminationFlags = EmissiveIsBlack`. Re-enable only with a proper mostly-black emissive map that lights just the intended detail. Note the in-editor preview can look fine while the in-game shader still applies emission, so verify in-game.

---

## Washed-out / pale custom HELD weapon (texture loads but looks bright & flat) → render it fullbright

A custom **held** weapon (game-rip / Quake-style view model) whose texture is verifiably correct in the bundle can still render **washed out and desaturated in daylight** (a pale ghost of the texture, only the brightest hues faintly showing) and **crushed to near-black at dusk/night**. The texture *is* sampling — the giveaway is a faint trace of the real colors and that the weapon is darker/brighter than a white surface would be. The cause is **7DTD's intense outdoor HDR sun**: `albedo × light` clamps toward white in bright sun (washing out all contrast) and toward black in low light. The Standard diffuse term has no exposure control to push back. (Quake-Weapons, June 2026.)

Two-stage history worth knowing so you don't stop early:
1. First suspected the Standard **glossy reflection + specular** (held items have no reflection probe, so they sample the bright default cube). Going matte — `_Metallic=0`, `_Glossiness=0`, `_GlossyReflections=0`+`_GLOSSYREFLECTIONS_OFF`, `_SpecularHighlights=0`+`_SPECULARHIGHLIGHTS_OFF` — removed a specular sheen but **did not fix the wash**: the plain diffuse still blew out in daylight and went dark at night. Necessary but not sufficient.
2. **The actual fix: render the weapon fullbright/unlit**, which is how Quake view models are authored anyway. Drive the surface from its own texture as **emission** and zero the lit albedo, so the texture shows at its true, constant colors regardless of time of day:
```csharp
m.color = Color.black;                       // kill the lit-albedo term (no wash, no crush)
m.SetTexture("_EmissionMap", tex);           // emission map = the real diffuse texture
m.EnableKeyword("_EMISSION");
m.SetColor("_EmissionColor", Color.white);   // 1.0 = texture shown at authored brightness
m.globalIlluminationFlags = MaterialGlobalIlluminationFlags.RealtimeEmissive; // NOT EmissiveIsBlack (that strips emission at build)
```
This is the deliberate, full-texture **inverse** of the placeholder-emission gotcha above: there a solid-colour mask map glowed a flat colour; here the emission map *is* the real diffuse, which is exactly what you want. Trade-off: a purely emissive model is flat (no light-driven form) and stays lit at night; for game-rip weapons that read as fullbright that's correct. If you want a little form back, keep a small dark albedo (`_Color` ~0.2–0.3 grey) on top of the emission, but any lit albedo can re-introduce some daylight wash, so keep it low.

Things that are NOT the cause (verified by cracking the bundle with `UnityPy`): texture pixel data (bundle mean RGB matched the source exactly), missing UVs, `m_ColorSpace` (albedo was `1`/sRGB, same as working mods), project color space (all these mods are Gamma `m_ActiveColorSpace: 0` and the world-entity ones render fine), shader stub (the bundled `Standard` had real compiled sub-programs), or material→texture wiring (all renderers resolved to textured mats). World-entity mods (vehicles/turrets/drones) don't show this because scene lighting on a world object is far less punishing than the first-person weapon-camera path on a held item.

### Selective glow on a fullbright weapon — id's `<base>.glow` additive maps

Quake view-model parts that glow (railgun coils/lens, etc.) come from a **separate `<base>.glow.<ext>` texture** sitting next to the diffuse (e.g. `railgun2.glow.jpg` beside `railgun2.tga`). It's a mostly-**black** additive map that's bright only where the part glows; id's weapon shader adds it on top of the lit diffuse. An MD3→OBJ converter **silently drops it**: the MD3 surface only stores *one* shader name (the base), nothing references the `.glow` sibling, and OBJ/MTL has no emission-map field Unity's importer reads. (Quake-Weapons, June 2026.)

Three different naming conventions, only one is a safe additive overlay:
- **`<base>.glow.<ext>`** (dot-glow) — the real additive glow companion. Add it. The *only* ones in the Q3 pak are `railgun2.glow` / `railgun3.glow`.
- **`<weapon>_glo` / `<weapon>_e`** (e.g. `plasma_glo.tga`, `bfg_e.TGA`) — a standalone energy texture that a surface uses **directly as its base shader**. It already renders bright in the fullbright path; do NOT also match it as a `<base>_glo` companion or you'll paint energy over the body. (This is why the companion search keys on the dot-`.glow` suffix only.)
- **`f_<weapon>`** (e.g. `f_machinegun.jpg`) — Quake Live "fullbright" skin: a *large bright* flare that assumes a dimly-lit base. On our already-fullbright emission base it blows the whole surface to white. Skip it.

Pipeline used (extends the fullbright recipe above): `tools/md3_to_obj.py` looks up the `<base>.glow` sibling, copies it next to the OBJ, and writes a `glow.txt` manifest of `<mat_name>=<glow_file>` lines (one entry per energy surface; absent for the ~12 weapons with no glow). `SetupQuakeWeapons.cs` reads it and, for those materials, bakes **emission = clamp01(diffuse + glow)** into a single `_EmissionMap` (Standard has only one emission slot, so the two passes must be combined) and sets `_EmissionColor = white × ~2.2` so only the glow pixels read bright/bloom while the matte body (black in the glow map) stays calm. Gotchas:
- The combined map must be saved as a **real `.png` asset** (`EncodeToPNG` → write → `ImportAsset` → load), exactly like the `.mat`-must-be-an-asset rule — an in-memory `new Texture2D` is lost when the bundle is built.
- Source textures need `TextureImporter.isReadable = true` so `GetPixelBilinear` works; sample at the larger of the two resolutions and bilinear-resample the smaller (diffuse and glow are often different sizes).
- A glow surface whose base texture is **absent from the pak** (railgun's coil — `railgun2.tga` doesn't exist, only its `.glow`) must stay base-less: skip the "grab any image in the folder" diffuse fallback when a glow map is present, or the combined emission picks up an unrelated body skin.

---

## Animating a baked emission map at runtime (recreate Quake `tcMod scroll`)

Quake 3 weapon energy (railgun coil, plasma/BFG glow) is an **animated** shader —
`tcMod scroll` streams the energy texture along the surface. The Standard shader
the bundle bakes to has **no UV animation**, so a baked `_EmissionMap` energy
stripe renders as a static painted-on band. You don't need a custom shader (those
get stripped → magenta) or a bundle rebuild to animate it: **scroll the material's
`_EmissionMap` texture offset every frame from a runtime MonoBehaviour**, exactly
like the held-item drivers (hum/beam/placement).

Recipe (Quake-Weapons `src/QuakeGlow.cs`, June 2026):
- Resolve the locally-held model the standard way — `world.GetPrimaryPlayer()` →
  `inventory.holdingItem` (name gate) → `inventory.GetHoldingItemTransform()`.
- Read `renderer.materials` (NOT `sharedMaterials`) so Unity instantiates
  per-renderer copies — the scroll then stays local and never mutates the shared
  bundle material asset. The instances die with the model, so the offset resets
  for free whenever the game re-clones the held item (view switch / hold-type
  change — see Items.md). If a renderer carries no energy surface, set
  `r.materials = r.sharedMaterials` back so you don't leak useless copies.
- **Identify the surface by a substring of its `_EmissionMap` texture name** —
  the bake preserves it (the diffuse texture name for a fullbright surface, e.g.
  `railgun4`; `<mat>_emis` for a baked combined diffuse+glow energy map). "Has an
  emission map" is NOT a discriminator (the fullbright trick gives *every* surface
  one) — match the specific name.
- **The animated surface is often a plain fullbright diffuse, NOT a baked glow
  map.** Quake-Weapons' first cut only scrolled the `_emis` coil/lens maps and the
  player saw no change — the band they meant (the railgun's top energy strip,
  `railgun4.jpg`) is a neutral-grey diffuse that Q3's *weapon shader* both
  **green-tinted** (`rgbGen`) and scrolled. The static bake kept it grey and
  still, so it read as a dead "white area on top." Recreating it needs BOTH:
  force `_EmissionColor` to the energy colour (HDR >1 on the dominant channel so
  it blooms) AND scroll the UVs. A scroll alone on an untinted/uniform region is
  invisible — a solid colour looks identical at any UV offset.
- Scroll with `m.SetTextureOffset("_EmissionMap", new Vector2(Mathf.Repeat(Time.time*sx,1f), Mathf.Repeat(Time.time*sy,1f)))`.
  Wrapping with `Mathf.Repeat(...,1f)` tiles seamlessly **only if the emission
  texture imports with Repeat wrap mode** (Unity default — don't set Clamp).
- Pick the scroll axis to match the coil's UV layout (railgun rings run along the
  barrel → scroll V). Caveat: a base+glow surface (railgun *lens*) scrolls its
  whole combined map too; if that looks wrong, name-match just the glow-only coil
  material rather than all `_emis` surfaces.

This is the runtime-animation complement to the static emission baking above:
bake the look in the bundle, animate the cheap part (a single UV offset) in C#.

---

## Importing a `.gltf` / `.glb` model (UnityGLTF) and the magenta-in-game gotcha

7DTD model bundles are usually FBX, but glTF/GLB (e.g. Blockbench or Sketchfab
exports) work via the **UnityGLTF** package (`org.khronos.unitygltf` from the
Khronos GitHub repo) in `UnityProject/Packages/manifest.json`. Reference mods:
`jigsaw` (a `.glb`), `Airstrike` (a self-contained `.gltf`). UnityGLTF registers
a ScriptedImporter for both extensions, so `AssetDatabase.ImportAsset(path,
ForceSynchronousImport)` then `LoadAssetAtPath<GameObject>(path)` yields the
model. A Blockbench `.gltf` is fully self-contained (geometry + all images are
base64 `data:` URIs in the one file) — you only need the `.gltf`, not the
extracted `textures/` siblings. Reference mod for the glTF flow: `jigsaw` (a
`.glb`). (Airstrike originally shipped a `.gltf` too but migrated to a native
OBJ import in 2026-06 — see the OBJ section below.)

**Critical gotcha — rebake materials onto `Standard`.** UnityGLTF assigns its
own `UnityGLTF/PBRGraph` shader. That shader does **not** exist inside 7DTD, so
a bundle shipped with it renders **bright magenta** in-game (shader-not-found
pink). Convert every imported material to Unity's built-in `Standard` shader in
the setup script before saving the prefab, copying base color + textures across:

```csharp
Material ToStandard(Material src) {
    var m = new Material(Shader.Find("Standard"));
    Texture albedo = FirstTexture(src, "_BaseColorTexture","baseColorTexture","_MainTex","_BaseMap")
                     ?? src.mainTexture;
    if (albedo) m.mainTexture = albedo;
    foreach (var pn in new[]{"_BaseColorFactor","_BaseColor","_Color"})
        if (src.HasProperty(pn)) { var c = src.GetColor(pn); c.a = 1f; m.color = c; break; }
    Texture n = FirstTexture(src, "_NormalTexture","normalTexture","_BumpMap");
    if (n) { m.SetTexture("_BumpMap", n); m.EnableKeyword("_NORMALMAP"); }
    return m;
}
```
Map old→new 1:1 and reassign `renderer.sharedMaterials`. `Standard` ships with
the game, so it always renders. (FBX imports with `materialImportMode=
ImportStandard` already use Standard and don't need this — it's a glTF-specific
trap.)

**Scale/units:** Blockbench glTF exports are already in metres — a low-poly A-10
came in at ~17.8 m wingspan (real ≈ 17.5 m), so import at `globalScale = 1`.
Still assert the renderer-bounds longest axis and log it.

**Axis/orientation:** the glTF→Unity conversion (right- to left-handed, Z flip)
makes the model's baked "forward" unpredictable headless. If a flying/driven
model points the wrong way in-game, correct it at runtime with a yaw-offset knob
(`LookRotation(travelDir) * Quaternion.Euler(0, yawOffset, 0)`, try 0/90/180/270)
rather than guessing in the bake. Parent the imported GLB under an identity-root
wrapper named "Visual" so the runtime drives a neutral transform.

Headless bake is identical to the FBX flow: tag the prefab with the bundle name,
`BuildPipeline.BuildAssetBundles(<mod>/Resources, ...)`, drive it from a
top-level (non-namespaced) `*HeadlessBuild.Run` via `-executeMethod`, and verify
the `Resources/*.unity3d` exists before reporting success.

---

## Importing an OBJ game-rip (Wraith / COD-Ghosts "iw6") — native Unity OBJ flow

No special package needed: Unity's built-in model importer handles `.obj`
directly (mesh + UVs + normals + per-`usemtl` submeshes). Reference mod:
`Airstrike` (the A-10 swapped from glTF to a Wraith-extracted OBJ, 2026-06).
Rips from **Wraith** (`# Generated by Wraith` header) carry several gotchas:

- **Units are centimetres, axis is Z-up.** Raw A-10 spanned ~1758 units across
  with the *smallest* span on Z (the ~430-unit vertical). Set
  `ModelImporter.useFileScale = false; globalScale = 0.01f` (cm→m), and parent the
  imported OBJ under an identity "Visual" child rotated `Euler(-90,0,0)` to take
  Z-up → Unity Y-up. Verify from the bake log that the **height** axis (y) is the
  small one (~4.3 m for an A-10) — if y is large the model is standing on its tail;
  flip the X rotation to `+90`. (Same single-source-of-truth sizing rule as above:
  size via `globalScale`, then *assert* bounds, never rescale from runtime bounds.)
- **The `.mtl` texture paths are broken** (`map_Kd ..\_images\foo.tga`) and point
  at `.tga` that may not ship. Don't rely on Unity auto-binding them. Set
  `materialImportMode = ModelImporterMaterialImportMode.ImportStandard` so each
  `usemtl` group becomes a named Standard material, then in the setup script
  **bind the textures explicitly by material name** — e.g. body vs glass keyed on
  `srcName.ToLower().Contains("glass")` — loading the PNG copies you placed in
  `Assets/.../Source/` via `AssetDatabase.LoadAssetAtPath<Texture2D>`. Mark normal
  maps `TextureImporterType.NormalMap` + `sRGBTexture=false` before building the
  material (or an OnPreprocessTexture postprocessor) so `_BumpMap` is valid, and
  set `importTangents = CalculateMikk` on the model so normal mapping works.
- **Nose direction for the yaw knob:** find the nose programmatically by comparing
  cross-section spread at each end of the long axis — the *narrow* end (small
  wingspan + height spread) is the nose. For the A-10 the nose was at +X while the
  runtime flies the prefab's +Z forward, so the heading needed `ModelYawOffset=90`
  in items.xml (tunable without a Unity rebuild; try 270/180/0 if it flies
  backward/sideways). Unlike FBX/glTF, OBJ imports use the Standard shader already,
  so no magenta-rebake is needed — but rebuilding clean Standard `.mat` assets is
  still required so they pack into the bundle as dependencies.

The `.rar` a rip ships in (Wraith bundles `.obj` + `.mtl` + `.ma`/`.smd`/`.mesh.ascii`
+ `.tga` textures); extract with `7z x`, copy just the `_LOD0.obj` and the PNG
textures into `UnityProject/Assets/.../Source/`, and delete the old source model +
generated prefab/material assets before re-baking so stale meshes don't get bundled.

---

## Entity-only explosions / "no block damage" area effects (`ExplosionServer`)

To damage entities in an area but never scratch terrain or builds (airstrikes,
energy weapons, etc.), use the stock explosion API with the block fields pinned
to zero — no custom per-entity loop needed:

```csharp
var props = new DynamicProperties();
props.Values["Explosion.ParticleIndex"]   = idx.ToString();   // WorldStaticData.prefabExplosions index
props.Values["Explosion.BlockDamage"]      = "0";             // <- no block damage
props.Values["Explosion.RadiusBlocks"]     = "0";             // <- no block scan
props.Values["Explosion.EntityDamage"]     = dmg.ToString(CultureInfo.InvariantCulture);
props.Values["Explosion.RadiusEntities"]   = radius.ToString(); // int
props.Values["Explosion.BlastPower"]       = "0";
props.Values["Explosion.DamageBonus.water"]= "0";
var data = new ExplosionData(props, null);
if (data.BuffActions == null) data.BuffActions = new List<string>();
GameManager.Instance.ExplosionServer(worldPos, World.worldToBlockPos(worldPos),
    Quaternion.identity, data, ownerEntityId, 0f, false, sourceItemValue);
```

> **API note (verified against installed Assembly-CSharp.dll, 2026-06):** current
> signature is `ExplosionServer(Vector3 _worldPos, Vector3i _blockPos, Quaternion
> _rotation, ExplosionData _explosionData, int _entityId, float _delay, bool
> _bRemoveBlockAtExplPosition, ItemValue _itemValueExplosionSource = null)` — **8
> args, no leading `clrIdx`**. A newer game build dropped the old leading `int
> clrIdx` from the whole explosion API family (`ExplosionServer` *and*
> `ExplosionClient`). Passing 9 args → `CS1501: No overload ... takes 9 arguments`.

`ExplosionServer` handles entity damage, VFX (`ParticleIndex`), explosion sound,
owner attribution (kills credit the player), and MP networking. With
`RadiusBlocks=0` the block-damage pass is skipped entirely. For a dense
"strafing line", march many of these along a path; to avoid hundreds of
overlapping explosion **sounds**, do real damage on a coarse spacing (~2–3 m)
and fill the gaps with cosmetic-only puffs: `Instantiate(WorldStaticData
.prefabExplosions[i].gameObject, pos - Origin.position, ...)` with their
`AudioSource`s muted/disabled and `Light`s disabled, `Destroy`ed after ~1 s.
Those prefabs are pure VFX — instantiating one applies no damage. Remember every
rendered world position needs `- Origin.position` (floating origin); the
`ExplosionServer` call itself takes the true world position. Reference:
`Airstrike` (`AirstrikeRunManager.cs`), modelled on `RocketTurret`.

### Timing a sustained strafing gun run (aircraft)

When the fire-front tracks a moving aircraft (rounds land `GunLead` metres ahead
of the nose), the **gun run paints a line as long as the plane travels while
firing**: `fireLength = burstSeconds * jetSpeed`. You can't get a multi-second
burst over a *short* corridor with a fast plane — the plane crosses 60 m in 0.4 s.
So pick the burst duration and let the visual line follow from it; keep *lethal*
damage gated to a central sub-window (`StrafeLength`) while the rest of the line is
cosmetic. To make the plane sit overhead the target at a fixed `T` seconds after
spawn (to sync an engine sound), spawn it `jetSpeed * T` back along the run.

For the cosmetic "walking rounds" along a long line, do **not** instantiate an
explosion prefab per round — dozens of particle systems will hitch. Use a cheap
billboard dust puff instead: one `GameObject.CreatePrimitive(Quad)` per round with
an alpha-blended dust texture, scale-up + alpha-fade over ~0.45 s via a tiny
MonoBehaviour, then self-destroy (`AirstrikeImpactPuff.cs`). Reserve the real
grenade explosion for the lethal central impacts only.
