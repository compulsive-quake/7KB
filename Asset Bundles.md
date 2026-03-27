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

---

## Gotchas

> **Missing T_Block tag** — If your ModelEntity block renders but can't be interacted with in-game, the prefab's root GameObject is likely missing the `T_Block` tag. Ensure the 7DTD TagManager.asset is installed and the root is tagged.

> **Bundle already loaded** — If the game loaded the bundle via a block `Model` property, `AssetBundle.LoadFromFile` will fail. Use `GetAllLoadedAssetBundles()` first.

> **Asset paths are lowercase** — Unity normalizes asset bundle paths to lowercase. Use case-insensitive matching when searching for assets.

> **Dedicated server** — Don't load visual assets (particles, audio) on dedicated servers. Guard with `if (GameManager.IsDedicatedServer) return;`

> **Bundle file location** — The game resolves `#@modfolder:` to the mod's root directory. Place bundles in `Resources/` by convention.
