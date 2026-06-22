# Items

Part of the [7DTD Modding Knowledgebase](README.md). Covers custom item definitions, icons, and properties.

---

## Explosion particle indices (`Explosion.ParticleIndex`)

`WorldStaticData.prefabExplosions` is a `Transform[100]` loaded at startup from
addressables `Prefabs/prefabExplosion{i}.prefab` (i = 0..99). An item's
`Explosion.ParticleIndex` picks which one its blast shows. Indices seen in stock
`items.xml`:

| Index | Used by |
|---|---|
| 13 | `thrownGrenade`, `thrownGrenadeContact` — clean grenade burst, **no vehicle debris** |
| 21 | `thrownTimedCharge` |
| 36 | rocket/big munitions |
| 7 | zombie vomit/boulder projectiles |
| 4 | vehicle explosion — throws **car-part debris** (avoid for generic blasts) |

**Best way — spawn the real grenade FX via `GameManager.ExplosionClient`.**
The cleanest "pure visual, no block damage" explosion is the game's own
client-side FX call — literally the path a thrown grenade runs for its visuals:
```csharp
// _clrIdx is never read inside ExplosionClient, so 0 is fine.
// It subtracts Origin.position itself — pass the FULL world pos.
// Empty change list => no block edits; 0 blastPower/radius => no knockback.
static readonly List<BlockChangeInfo> NoChanges = new List<BlockChangeInfo>();
GameManager.Instance.ExplosionClient(0, worldPos, Quaternion.identity,
    index, 0, 0f, 0f, entityId, NoChanges);
```
`ExplosionClient` only edits blocks when handed a **non-empty** change list, so
an always-empty list guarantees zero terrain damage. It plays the full burst
(fireball, light flash, lingering smoke, ground scorch) and lets the prefab's
own `TemporaryObject` end it at the natural duration — no manual `Destroy`
timer, so the effect is never cut short. (Airstrike routes both its impact and
cosmetic FX through this — fixed June 2026.)

**Why not raw `Instantiate`** (the older Airstrike approach): two gotchas.
1. **Magenta quad on the ground.** Some indices are broken/missing-material
   prefabs (Airstrike's cosmetic puffs used **index 1**, which drew a magenta
   "missing material" quad below the blast). Stick to known-good indices — **13
   is the safe thrown-grenade burst**. (Vehicle indices 4/29 also bake in a
   debris/carcass mesh you usually don't want.)
2. **Don't truncate the lifetime.** A short `Destroy(go, 1.1f)` cuts the burst
   off so it "disappears quickly" vs a real grenade. Let the prefab's
   `TemporaryObject` self-time it; if you must cap, keep it well past the
   natural duration (~3–4s).

**Perf — don't spawn many of these at once.** `ExplosionClient` (and raw
`Instantiate`) of a grenade prefab is *expensive*: each is a full particle system
+ a real-time dynamic light + a scorch decal. A strafing run that fired ~24 of
them within ~0.4s tanked FPS hard. For dense/rapid impacts, reserve real
explosion prefabs for a handful of hero blasts (or none) and use **cheap billboard
puffs** instead — one camera-facing quad with a procedural radial texture that
scales up + fades over ~0.5s, no light, no particle system. Additive shader =
fire, alpha-blended = dust/smoke. See `Airstrike/src/AirstrikeImpactPuff.cs`
(`SpawnFire`/`Spawn`); Airstrike dropped the per-impact grenade burst for these in
June 2026 specifically to fix the FPS melt.

---

## Defining Items

Items are defined via XPath patches in `Config/items.xml`:

```xml
<configs>
    <append xpath="/items">
        <item name="myCustomItem">
            <property name="Meshfile" value="Items/Misc/oilGP"/>
            <property name="DropMeshfile" value="Items/Misc/sacPT"/>
            <property name="CustomIcon" value="myCustomItem"/>
            <property name="CustomIconTint" value="255,255,255"/>
            <property name="Stacknumber" value="64"/>
            <property name="CreativeMode" value="Player"/>
            <property name="Group" value="Decor/Miscellaneous"/>
            <property name="DescriptionKey" value="myCustomItem_desc"/>
            <property name="Material" value="Mplastics"/>
        </item>
    </append>
</configs>
```

> **`Meshfile` is required** — omitting it causes an exception during item loading and the item will not be registered.

Items can also inherit from existing items using `Extends`:

```xml
<item name="myItem">
    <property name="Extends" value="resourceWood"/>
    <property name="CustomIcon" value="myItem"/>
</item>
```

> **Caution with `Extends`:** Vanilla item and block names can change between game versions. If the parent name doesn't exist, the XML parser throws `Could not find Extends block/item X` and the entire file fails to load. Prefer defining items from scratch with explicit properties rather than extending vanilla names that may not exist in the target game version.

---

## Item Properties Reference

| Property | Example value | Notes |
|---|---|---|
| `Meshfile` | `"Items/Misc/oilGP"` | **Required.** 3D mesh when held. Use `Items/Misc/oilGP` for generic small items |
| `DropMeshfile` | `"Items/Misc/sacPT"` | 3D mesh when dropped on ground |
| `CustomIcon` | `"myItem"` | Icon name — maps to `UIAtlases/ItemIconAtlas/{value}.png` |
| `CustomIconTint` | `"255,255,255"` | RGB tint applied to the icon (white = no tint) |
| `Stacknumber` | `"64"` | Max stack size in inventory |
| `CreativeMode` | `"Player"` | Makes the item appear in creative menu. Use `"Dev"` for admin-only |
| `Group` | `"Decor/Miscellaneous"` | Category in creative menu (e.g. `"Ammo/Weapons"`, `"Food/Cooking"`, `"Tools/Traps"`) |
| `DescriptionKey` | `"myItem_desc"` | Localization key for the item description |
| `Material` | `"Mplastics"` | Material type (affects sounds, physics) |
| `Extends` | `"resourceWood"` | Inherit all properties from another item, then override specific ones |
| `Weight` | `"5"` | Item weight |
| `EconomicValue` | `"100"` | Trader value |
| `HoldType` | `"45"` | Animation type when held |

---

## Custom 3D Meshes (Asset Bundles)

The `Meshfile` and `DropMeshfile` properties can reference Unity asset bundles using the `#@modfolder:` syntax:

```xml
<property name="Meshfile" value="#@modfolder:Resources/mybundle.unity3d?Assets/MyItem/MyPrefab.prefab"/>
<property name="DropMeshfile" value="#@modfolder:Resources/mybundle.unity3d?Assets/MyItem/MyPrefab.prefab"/>
```

The syntax is: `#@modfolder:Resources/{bundleName}?{assetPath}`

### Building Asset Bundles

Create a Unity project (must match the game's Unity version) with an Editor script:

```csharp
[MenuItem("MyMod/Build Asset Bundle")]
public static void Build()
{
    var builds = new AssetBundleBuild[1];
    builds[0].assetBundleName = "mybundle.unity3d";
    builds[0].assetNames = new string[] { "Assets/MyItem/MyPrefab.prefab" };

    BuildPipeline.BuildAssetBundles(outputPath, builds,
        BuildAssetBundleOptions.None, BuildTarget.StandaloneWindows64);
}
```

Place the built `.unity3d` file in `Resources/` within your mod folder.

### Held Item Scale & Positioning

There is **no XML property** to control the scale or position of an item when held in the player's hand. The held item's transform (position, rotation, scale) is determined by:

1. The root transform scale baked into the Unity prefab
2. The game's internal positioning system based on `HoldType`

To adjust hand scale, modify the prefab's root `localScale` in Unity and rebuild the asset bundle.

> **Runtime note:** The game re-applies the held item's transform every frame during `Update`. Any code modifying the held item transform must do so in `LateUpdate` to apply after the game's positioning pass. Offsets should be applied *relative* to the game's values (add to position/rotation, multiply scale) rather than setting absolute values, since the game sets its own base values each frame.

> **Strongly prefer authoring the prefab correctly over runtime transform adjustment.** The cleanest held item needs **zero** runtime transform code: bake the right pivot/scale into the prefab, put it on the `HoldingItem` layer (see below), and pick a `HoldType` in `items.xml`. Runtime "hand offset tuner" code that pokes the live held-item transform every frame is a recurring source of pain (it was ripped out of RocketTurret once, re-added strictly scoped for zPhone live tuning, then removed again in June 2026 once values were finalized — see the end-state pattern below):
> - Poking the held **root** with a cache to recover the game's base can latch onto a **world** position (the game doesn't reliably re-seat the root every frame for every setup) — the model then freezes in space while the player walks away (the laser/aim keeps tracking off the head vector, masking the cause).
> - Even offsetting the model's **children** instead is risky: if the engine reuses a shared held-item holder GameObject across items, your modifications can bleed into **other mods'** held items (their model shows up scaled/displaced/invisible). A user reported exactly this — "other apps' held items stopped showing when this mod is installed."
> - **Shrink-on-reequip:** the game can hand back the **same** held transform when the player switches away and back to your item. If your tuner left its multiplied scale / added offsets baked in, the re-acquisition captures that modified pose as the new "base" and applies the tuning again — with a scale multiplier < 1 the item gets smaller on every item switch. Fix: when your item stops being held, **restore the captured base transform** (pos/rot/scale) before dropping your cached reference, so the next acquisition starts from pristine values (see `ReleaseHeldModel()` in RocketTurret).
>
> If you genuinely need an offset, do it in the prefab, not at runtime. If you must do it at runtime, scope it so it only ever runs while *your* item is held and only touches *your* freshly-instantiated model, and treat any interaction with the shared held-item transform as suspect.
>
> **End-state pattern for hardcoded runtime placement (RocketTurret, then FPV, June 2026):** once tuning is done and the live-tuner app is removed, don't write the transform every frame. Apply the placement **once** when a fresh held transform is detected (same trigger as the relayer), then keep a **compare-only watchdog** in `LateUpdate`: if the model's pose differs from the last values you wrote (position/scale changed, or rotation differs by > 0.01°), the game repositioned it — recapture its new pose as the base and re-apply on top. Steady state does zero transform writes; the watchdog exists because the game's Update-pass repositioning is real but not every-frame for every setup. A pure one-shot with no watchdog risks being stomped on HoldType refreshes / view changes. Still restore the captured base on holster (shrink-on-reequip, above). Note the prefab-bake alternative isn't a drop-in for tuned runtime values: the runtime offset is applied in the held root's *parent* space (`base.pos + offset`), so moving it into a prefab child node changes the composition (child offsets get rotated/scaled by the base) and needs a retune. Scale variant (FPV): if the tuner sized the model via a bounds-based normalizer (measure world renderer bounds, multiply `localScale` to hit a target visible size), that call is idempotent and safe to keep in the once-per-equip apply — bake `targetSize * tunedHandScale` into one constant instead of converting it to a raw scale multiplier.

> **Custom weapon that `Extends` a vanilla gun NREs in the toolbelt the instant you select it → missing `Attachments/Scope` node in the prefab.** Nearly every vanilla ranged weapon has `Action1` with `Class="Zoom"` (`ItemActionZoom`, the ADS/iron-sights action), and a custom gun inherits it via `Extends`. The instant the held model is built (selecting the item in the toolbelt → `Inventory.SetItem` → `CreateInventoryData` → `ItemActionZoom.CreateModifierData`), the `ItemActionDataZoom` **constructor** runs this against the held prefab:
> ```csharp
> Scope = model.FindInChilds("Attachments");
> if (Scope == null) Log.Error("Transform 'Attachments' not found ...");   // graceful
> else { Scope = Scope.Find("Scope"); HasScope = Scope.childCount > 0; }    // NRE if no "Scope" child
> ```
> So there are **two** requirements, and missing *either* behaves differently: no `Attachments` node at all → just a logged error (no crash); an `Attachments` node **without** a `Scope` child → `Find("Scope")` returns null and `.childCount` throws `NullReferenceException` (stack: `ItemActionDataZoom..ctor` → `CreateModifierData` → `ItemClass.CreateInventoryData` → `Inventory.SetItem` → `XUiC_Toolbelt.UpdateBackend`). `OnModificationsChanged` repeats the same unguarded `Attachments`→`Scope`→`childCount` walk, so catching only the ctor isn't enough — **the fix must be in the prefab.** Bake an empty `Attachments` GameObject under the prefab root **and an empty `Scope` child under it** (vanilla weapon rigs always carry `Attachments/Scope`). `Scope.childCount == 0` → `HasScope=false` → no optic, which is what a scope-less custom gun wants. No XML-only workaround: `Extends` resolves the inherited `Action1` at the `ItemClass` level, not as a DOM child, so you can't `<remove>` the zoom action by xpath on the child item. (Quake-Weapons `SetupQuakeWeapons.cs`, June 2026 — the build script originally baked only `Attachments` and still NRE'd until the `Scope` child was added.)
>
> **"I'm holding it but the hand looks empty" gotcha:** First decide whether it's invisible in *both* views or *only first person* — they have different causes.
>
> **Invisible in first person but VISIBLE in third person → wrong layer (most common for custom asset-bundle meshes).** 7DTD has a dedicated **`HoldingItem`** layer (confirmed in the game's layer table, alongside `Shadow`, `RenderInTexture`, `NGUI`). Vanilla held items live on this layer permanently; **both** the main camera (third person) and the first-person weapon camera (which draws the held item on top so it never clips through walls) render it. A custom bundle mesh comes in on its baked-in `Default` layer, which only the main camera draws → shows in third person, invisible in first. **The game does NOT relayer custom-attached models** (the tell: in third person the custom item is still sitting on `Default`, not `HoldingItem` — if the game relayered it, it wouldn't be on `Default`). So set the layer yourself the way the game authors its own items: put the whole held hierarchy on `HoldingItem`, **once**, when the model is instantiated — `int l = LayerMask.NameToLayer("HoldingItem"); foreach (Transform t in held.GetComponentsInChildren<Transform>(true)) t.gameObject.layer = l;`. Resolve by name (don't hard-code the index). No per-frame work and no first/third-person detection are needed — that was a wrong turn; propagating the *root's* layer each frame does nothing because the root stays `Default`. Trigger the one-time set off "held transform changed" in the `LateUpdate` hand tuner (a model refresh re-instantiates the transform, so this re-applies automatically). Ideally bake the layer into the prefab at build time instead, but runtime-by-name avoids depending on the Unity project's layer indices matching the game's.
>
> **World-space effects (laser beams, muzzle flashes) anchored to a held-item child transform appear visually detached from the rendered model in first person — even when the transform data is verified correct end-to-end.** Cause: held items on the `HoldingItem` layer are drawn by UFPS's weapon camera (7DTD ships UFPS — `vp_FPCamera` / `vp_FPWeapon` with its own rendering FOV, see `originalRenderingFieldOfView` in Assembly-CSharp), while a world-space `LineRenderer` on `Default` is drawn by the world camera (`EntityPlayerLocal.playerCamera`). Different projections → the muzzle's true world position lands on a different screen pixel than the rendered aperture, so the beam start floats off the device (typically toward screen center). Debugging tell: prefab marker position, bundle freshness, and `FindDeepChild` lookup all check out, yet the beam is offset. Fix (RocketTurret, June 2026): screen-space re-anchor — project the muzzle through the weapon camera (`WorldToViewportPoint`), unproject through the world camera (`ViewportToWorldPoint`) at the same view depth clamped past the world camera's near plane; find the weapon camera at runtime as the enabled camera ≠ world camera whose `cullingMask` includes `HoldingItem` (highest depth wins), and fall back to the raw muzzle position when none is active (third person). Don't instead put the LineRenderer on `HoldingItem` — the far end must hit a world target and would be drawn on top of geometry with the wrong projection.
>
> **Better resolution (RocketTurret, June 2026 — what actually shipped): don't anchor the beam to a model pixel at all.** The screen-space reprojection above technically works but is fragile and was abandoned. The robust pattern (ported from the Airstrike designator, which uses the same vanilla flashlight model) is: base the beam start on the held model's **transform position** (`inventory.GetHoldingItemTransform().position`, falling back to `cameraTransform.position`) plus a small offset built from the **camera basis** (`cam.right*R + cam.up*U + cam.forward*F`), with R/U/F tuned live by numpad. Cast the *painting* ray from the **camera center** (`cameraTransform.position/forward`), not the model, so the endpoint sits under the crosshair at every pitch. You're no longer trying to pin the start to a specific rendered pixel of the model, so the two-camera FOV mismatch stops mattering — the small camera-space offset reads as "emerging from the device" without pixel-exact registration. Draw it per-frame in a `LateUpdate` `MonoBehaviour` on the laser GameObject (after the camera moves) rather than in `ItemAction.OnHoldingUpdate` (throttled tick → laggy/smeared beam). Both model and camera transforms are render-space, so add `Origin.position` to express the start in un-shifted world coords (and subtract it again in `LineRenderer.SetPosition`). This also dropped the whole custom-mesh + `HoldingItem`-relayer + hand-placement-tuner stack: switching the item's `Meshfile` to a vanilla prefab (`@:Other/Items/Tools/flashlight02Prefab.prefab`) means the game handles layer/placement and none of the above first-person-invisibility gotchas apply.
>
> **Two follow-on bugs in that pattern, both fixed June 2026 — guard the beam start and don't poll global input:**
> 1. **Beam explodes into a screen-filling wedge after another mod switches the view (e.g. flying the FPV drone).** A world-space `LineRenderer` whose **near endpoint sits at or behind the camera near-plane** projects into a giant triangle (narrow far end, splayed near end). Cause: a first→third→first view switch (`EntityPlayerLocal.SetFirstPersonView`, which FPV does on enter/exit) churns the held-item rig and can leave `GetHoldingItemTransform().position` parked on the **player's body — i.e. on top of the lens** — so the model-based start lands at the camera. The *visual* held model can still look fine while the returned transform is bad. Fix on the **consumer (laser) side**, not by trusting the other mod to restore the transform. Two things, the first being the actual root cause:
>    - **Build the beam from the LIVE camera, not `EntityAlive.cameraTransform`.** That field is a *cached* transform the view switch can leave parked in the third-person pose while the real first-person camera (the one actually rendering) is correct. Use `EntityPlayerLocal.playerCamera` (the live `Camera`) — fall back to `Camera.main`, then the cached `cameraTransform`. Use it for BOTH the laser origin basis and the crosshair raycast. A start computed from the parked transform is ~0.4m in front of the *parked* camera but lands behind the *real* camera's near plane → cone. (A `Dot(model.position - cam.position, cam.forward) >= 0.25f` guard against a model parked at the camera is also worth keeping, but it does nothing if `cam` itself is the stale one — which was the trap: the first attempt guarded against the stale transform and the cone persisted.)
>    - **Clamp the near endpoint to the real camera's near plane as a guaranteed backstop.** Regardless of root cause, a LineRenderer only explodes when an endpoint is behind the near plane, so never hand it one: project both endpoints onto the camera forward axis; if the start's view-z is below `nearClipPlane + margin`, slide it forward *along the beam* to that plane; if the target is also behind, hide the line entirely. This kills the visual cone even if some other upstream transform is stale. Any mod drawing a world-space beam from a held item is exposed whenever a view-switching mod is installed — harden the consumer.
> 2. **A per-frame numpad live-tuner polling GLOBAL `Input` is a footgun — bake the values and delete it.** The Airstrike-style origin tuner (`Input.GetKeyDown` on arrows/keypad every frame from the laser's `LateUpdate`, mutating **unclamped** static fields) keeps firing even when the pointer isn't the focus, so a stray keypress or another mod grabbing input can shove the origin/width to absurd values and leave the beam permanently stretched. Once the offsets are final, promote them to `const` and remove the input poller (also drops the `UnityEngine.InputLegacyModule` reference). Same "tuners are scaffolding — bake and delete" rule as the hand/zPhone tuners.

> **Custom weapon prefabs MUST have a child Transform named `Attachments`** when they extend (or otherwise resolve to) an `ItemActionZoom`-using vanilla weapon (rifles, ARs, SMGs, pump shotguns — basically anything that can take mods). The first time the player draws the item, `ItemActionZoom.OnModificationsChanged` walks `_data.invData.model.FindChild("Attachments")` to parent any equipped mod GameObjects under, and if the lookup is null it logs `ERR Transform 'Attachments' not found in weapon prefab for <Name>(Clone).` followed by a `NullReferenceException at ItemActionZoom.OnModificationsChanged` that nukes the toolbelt XUi update (`ERR [XUi] Error while updating window group 'toolbelt'`). **This fires even when zero mods are equipped** — the parent transform must just exist. Fix: in your Unity bundle setup, add an empty GameObject named exactly `Attachments` as a direct child of the prefab root before SaveAsPrefabAsset. No mesh, no components — just the transform. Quake Weapons' `SetupQuakeWeapons.cs` does this; copy that pattern. Same likely applies to other vanilla weapon-script lookups (`Muzzle`, etc.) but `Attachments` is the one that fires immediately on draw.

> **Invisible in BOTH views → placement/pivot.** (1) The FBX geometry is offset from its own pivot, and that offset is *multiplied* by any visible-size normalization scale baked into the prefab (a 0.5m model auto-scaled ~3.7x turns a small modeling offset into a meters-large displacement) — re-center the mesh so its renderer-bounds center sits on the wrapper origin during the build. (2) It needs per-item offset tuning; if the mod ships a runtime hand-offset/HoldType tuner, confirm its `Register()` is actually **called** from `InitMod` (a defined-but-never-invoked registration leaves it stuck at the invisible default with no way to adjust). Verify the laser/muzzle child via `invData.model` to prove the mesh instantiated even when nothing is visible.

> **Making a gun-based item *feel* like a continuous beam (no custom ItemAction needed).** A fast hitscan-every-50ms weapon (high `RoundsPerMinute`) built on a vanilla full-auto gun reads as "chattering machine gun," not a steady beam — even with a beam VFX and continuous sound — because of inherited gun *feel*. You don't need a custom continuous-fire action class (7DTD has no native continuous-ranged weapon to model on, so that's a big C# build); zero out the discrete tells in `items.xml` instead. Model on vanilla **`gunToolDiggerAdmin`**: `KickDegreesVertical/HorizontalMin/Max = 0` (no camera recoil — the biggest tell at 1200 RPM, ~20 kicks/sec), `SpreadDegreesVertical/Horizontal = 0` + `IncrementalSpreadMultiplier = 0.001` (pinpoint, no bloom — also keeps the hitscan aligned with a beam VFX that's drawn from camera-center). Then blank `Particles_muzzle_fire value=""` (drop the gun muzzle smoke/flash; the beam is the visual) and enlarge `MagazineSize` + shorten `Reload_time` so the stream isn't broken by frequent mag swaps. All restart-bucket config; combined with a camera-center beam VFX and a held-trigger (`bPressed`) sound loop (see 7KB/Player Feedback Sounds.md), it sells "continuous beam" without touching mechanics code. The model's own fire *animation* (HoldType) may still play per shot, but with camera kick at 0 it's barely perceptible. (Quake-Weapons lightning gun, June 2026.)

---

## Item Icons (ItemIconAtlas)

Place a PNG file in `UIAtlases/ItemIconAtlas/` with a filename matching the `CustomIcon` property value:

```
UIAtlases/
└── ItemIconAtlas/
    ├── myCustomItem.png      ← matches CustomIcon="myCustomItem"
    └── anotherItem.png
```

The game **auto-discovers** these by filename — no C# registration needed. This is different from radial menu icons (`UIAtlas/`), which require runtime injection via `MultiSourceAtlasManager.AddAtlas()`.

### Gotcha: rendering icons from skinned-mesh prefabs

When auto-generating an item/block icon by rendering a prefab to PNG in Unity (e.g. an `ExportPrefabIcons` editor script with an off-screen camera + bounds-fit framing), a **rigged/armatured FBX** — i.e. one imported as a `SkinnedMeshRenderer` (`animationType` not `None`) — produces a **blank or tiny-speck icon**. Tell-tale sign: Unity's own Project-window thumbnail for that prefab is **blank grey** too, while a static-mesh prefab in the same project previews fine.

Cause: both Unity's `AssetPreview` thumbnail generator and a custom bounds-fit exporter frame the camera using the renderer's bounds. A `SkinnedMeshRenderer`'s root-bone AABB is **unreliable in edit mode** — for rigged FBXs it often reports wildly oversized bounds (one turret reported ~240 m vs. a real ~1.8 m model), so the camera frames mostly empty space and the model collapses to a dot. (`BakeMesh` is not a reliable workaround: depending on where scale lives in the hierarchy it can report ~0.02 m instead.)

Fix (one change fixes both Unity's thumbnail and the exporter): set **`m_UpdateWhenOffscreen: 1`** on the `SkinnedMeshRenderer` (inspector: "Update When Offscreen"). This forces Unity to recompute bounds from the actual posed vertices every render — respecting the full bone/scale hierarchy — so framing is correct. When the renderer comes from a nested model-prefab/FBX instance, apply it as a prefab override targeting the renderer's `fileID` (the same `fileID` whose `m_Materials` the prefab already overrides), `propertyPath: m_UpdateWhenOffscreen`, `value: 1`. No guessed bounds numbers needed (guessing wrong also breaks in-game frustum culling). Negligible per-frame cost for a single block/item.

Separately, a skinned mesh may still render **magenta** in the off-screen/export camera even after framing is fixed (it renders correctly in the Scene view) — that's a distinct render-path issue, not the bounds problem. Suspect batch/headless rendering of skinned meshes if the export runs via Unity `-batchmode`; rendering from the open editor (GPU present) is more reliable.

---

## Localization

Each item needs at least a name entry and optionally a description entry in `Config/Localization.txt`:

```
Key,File,Type,UsedInMainMenu,NoTranslate,english
myCustomItem,items,Item,FALSE,FALSE,My Custom Item
myCustomItem_desc,items,Item,FALSE,FALSE,A custom item for my mod.
```

The item name key must match the `name` attribute in `items.xml`. The description key must match the `DescriptionKey` property value.

---

## Creative Menu & Search

- Items appear in the creative menu when `CreativeMode` is set to `"Player"` (or `"Dev"` for admin-only)
- The `Group` property determines which category tab the item appears under
- The creative menu search filters items by their localized display name
- Items without a `Group` may not appear in category browsing but are still findable via search

---

## Dynamic Item Generation

Items are loaded from XML at game startup. Since `InitMod()` runs **before** Config XMLs are parsed (see [Mod Structure - Loading Order](Mod%20Structure.md#loading-order)), mods can dynamically generate `items.xml` at runtime:

```csharp
public void InitMod(Mod _modInstance)
{
    string configDir = Path.Combine(_modInstance.Path, "Config");

    // Generate items.xml based on runtime conditions
    // (e.g., scanning a folder for content)
    string xml = BuildItemsXml();
    File.WriteAllText(Path.Combine(configDir, "items.xml"), xml);
}
```

This technique is useful when item definitions depend on user-provided content (e.g., ROM files, custom textures) that isn't known at build time. The generated XML is picked up by the normal config loading pipeline.

Similarly, localization entries can be appended to `Config/Localization.txt` during `InitMod()` before the game reads it.

---

## Editor-Friendly Items.xml Model

When building tooling around `Config/items.xml`, treat an item as a mix of known structured fields plus preserved raw XML. The common modlet form is an XPath patch document:

```xml
<configs>
    <append xpath="/items">
        <item name="myCustomItem">
            <property name="Meshfile" value="Items/Misc/oilGP" />
            <property name="Stacknumber" value="64" />
        </item>
    </append>
</configs>
```

Some sources and generated files may instead use a full `<items>` root. A safe editor should support both, but new mod items are usually best written as `<append xpath="/items">` patches so they merge cleanly with vanilla XML.

### Core Field Meanings

| Field | XML | Meaning |
|---|---|---|
| Internal name | `<item name="...">` | Unique item ID used by recipes, loot, effects, console commands, icons, localization, and XPath selectors. |
| Extends | `Extends` | Inherits another item, then overrides selected properties. Convenient, but fragile when vanilla names change. |
| Tags | `Tags` | Comma-separated labels used by crafting, skills, repair/mod rules, actions, buffs, search, and item modifiers. |
| Group | `Group` | Creative/crafting category grouping. Items without a useful group may only be discoverable by search. |
| Display type | `DisplayType` | UI presentation template for inventory and tooltip display. |
| Description key | `DescriptionKey` | Localization key for the item's description text. |
| Class | `Class` | Engine item behavior class, such as food, medical, weapon, armor, quest item, or generic item. |
| Material | `Material` | Material definition used for sounds, physics, damage behavior, and some interactions. |
| Custom icon | `CustomIcon` | Uses another item/icon name for the inventory icon. If omitted, the game looks for an icon matching the item name. |
| Custom icon tint | `CustomIconTint` | RGB/accepted color tint applied to the inventory icon. White means no visible tint. |
| Mesh file | `Meshfile` | Held/world model path. For custom assets, this can point into an asset bundle with `#@modfolder:` syntax. |
| Drop mesh file | `DropMeshfile` | Model used when the item is dropped in the world. |
| Hold type | `HoldType` | Numeric animation/holding style used by the player model. |
| Stack number | `Stacknumber` | Maximum inventory stack size. |
| Economic value | `EconomicValue` | Base trader value before quality, perks, and modifiers. |
| Sellable to trader | `SellableToTrader` | Whether traders can buy this item. |
| Creative mode | `CreativeMode` | Controls creative-menu visibility, commonly player-visible or developer/admin-only. |

### Runtime HoldType Editing

When changing an item's `HoldType` at runtime from C#, do not assume `ItemClass.HoldType` is a plain integer or enum. In 7DTD v2.6 it may be stored as a data wrapper with a writable `Value` property; working runtime editors should read/write that nested `Value` before trying to replace the field/property itself. The public `EHoldType` enum is incomplete for XML tuning: the game accepts the animation/offset table indices used by vanilla `items.xml` rather than only the few enum names.

If HoldType reflection is driven from a held-item `LateUpdate()` tuner, wrap the write path defensively and throttle any warning logs. A failed `Convert.ChangeType` or wrapper write can otherwise throw once per frame while the item is held, flooding the console and making the item feel broken even though the XML loaded.

**Refreshing the held model after a live HoldType change — use `Inventory.ForceHoldingItemUpdate()`, NOT `SetRightHandAsModel()`.** After writing the new HoldType into the `ItemClass`, the currently-held item must be rebuilt to pick it up. `Inventory.SetRightHandAsModel()` looks tempting but is a trap: it sets `models[m_HoldingItemIdx] = avatarController.GetRightHandTransform()`, i.e. it overwrites the held-item slot with the bare right-hand *bone* transform. After that, `GetHoldingItemTransform()` returns the hand bone instead of your custom mesh, the custom mesh is orphaned, and a custom-bundle pointer **disappears in first person and never comes back** even when you switch HoldType back to a known-good value — nothing re-creates the real model until a full weapon swap. The correct call is `inv.ForceHoldingItemUpdate()`: it `Destroy`s the old `models[idx]` and re-runs `createHeldItem` (→ `CloneModel`), producing a **fresh transform**. A `LateUpdate` layer-tuner keyed on "held transform changed" then re-relayers the new mesh onto `HoldingItem` automatically, so the item reappears correctly. (Seen in RocketTurret: the zPhone "cycle hold type" action made the laser pointer vanish for good until `SetRightHandAsModel()` was replaced with `ForceHoldingItemUpdate()`.)

### Durability, Quality, And Repair

| Field | XML | Meaning |
|---|---|---|
| Degradation max | `DegradationMax` | Durability range or maximum durability, often quality-scaled. |
| Degradation per use | `DegradationPerUse` | Durability lost per use/action. |
| Degradation breaks after | `DegradationBreaksAfter` | Whether the item breaks at zero durability. |
| Quality tier | `QualityTier` | Quality/progression bucket used by generated item stats. |
| Mod slots | `ModSlots` | Number of modifier slots, commonly quality-scaled. |
| Repair tools | `RepairTools` | Items required or consumed to repair this item. |
| Repair amount | `RepairAmount` | Durability restored by a repair action or repair ingredient. |

### Actions And Effects

Items often contain nested action and effect data that should not be flattened too aggressively:

```xml
<property class="Action0">
    <property name="Class" value="Ranged" />
    <property name="Magazine_size" value="12" />
    <property name="Magazine_items" value="ammo9mmBulletBall" />
    <property name="Reload_time" value="2" />
</property>

<effect_group name="myItem">
    <passive_effect name="DamageEntity" operation="base_set" value="30" />
    <triggered_effect trigger="onSelfPrimaryActionEnd" action="AddBuff" buff="buffExample" />
</effect_group>
```

Common action fields include `Action0`/`Action1` class, `Auto_fire`, `Magazine_size`, `Magazine_items`, `Reload_time`, `Range`, `DamageEntity`, `DamageBlock`, `AttacksPerMinute`, `StaminaUsage`, `Projectile`, and `CrosshairOnAim`.

Common effect nodes include `effect_group`, `passive_effect`, `triggered_effect`, and `requirement`. Preserve unknown attributes and child nodes because many item behaviors are implemented by specialized action classes or version-specific effect fields.

### Custom Consumable Actions

For custom C# item actions that consume the held item, vanilla consumables use `holdingEntity.inventory.DecHoldingItem(1)` after the action successfully completes. `Inventory.DecHoldingItem` only decrements the count when the item can stack; if the item is non-stackable it clears the held slot. Set `Stacknumber` above `1` for grenade-like consumables that should decrement one item from a stack.

### Held Item Models vs First/Third-Person Switching

Mod code that switches the local player between first and third person (e.g. via `EntityPlayerLocal.SetFirstPersonView(bool, bool)`) churns the held item model heavily:

- `SetFirstPersonView` → `EModelSDCS.SwitchModelAndView` → `GenerateMeshes()` rebuilds the SDCS rigs (`SDCSUtils.CreateVizTP/CreateVizFP`), destroying rig children except `Origin`/`IKRig`.
- `AvatarSDCSController.SwitchModelAndView` then reparents the held item model (`rightHandItemTransform`) onto the new rig's `RightWeapon` bone and applies the **third-person gunjoint offsets even when switching back to first person** — the held model's local position/rotation are not what they were before the cycle.
- `EntityAlive.ShowHoldingItem(bool)` → `Inventory.ShowRightHand(bool)` just toggles the `RightWeapon` bone GameObject active. Pairing hide/show across a view switch is brittle: the bone instance/state can change in between, leaving the held item invisible or misplaced. Avoid hiding the held item across FP/TP switches.
- Recovery sledgehammer: `Inventory.ForceHoldingItemUpdate()` destroys and re-clones the held model with correct parenting/offsets. **Danger:** when the player holds bare hands, `models[holdingItemIdx]` can be the right-hand bone itself (`SetRightHandAsModel`) — `ForceHoldingItemUpdate` would destroy the bone. Only call it when a real holdable item is in the slot.
- `Inventory.SetRightHandAsModel()` is NOT a refresh: it just overwrites `models[holdingItemIdx]` with the right-hand bone transform (intended for bare hands / NPC setup). Calling it on a local player holding a real item corrupts the slot — the held model is never rebuilt, and any code that treats `GetHoldingItemTransform()` as the item model (offset/scale adjusters) starts mutating the hand bone, leaving the held item invisible/misplaced. To apply a runtime `HoldType` change to the held model, write the new value to the ItemClass and call `ForceHoldingItemUpdate()` (with the real-item guard above).
- `Inventory.DecHoldingItem` → `updateHoldingItem()` early-returns when the held ItemValue is unchanged (stack count is not part of the comparison), so decrementing a stack does not refresh held-item visibility/transform.

### Placement Items

Items that place blocks usually point back into `blocks.xml`:

| Field | XML | Meaning |
|---|---|---|
| Place as block | `PlaceAsBlock` | Block created when this item is placed into the world. |
| Preview block | `PreviewBlock` | Block/model used for placement preview. |
| Allowed placement | `Allowed_placement` | Rules for where and how the item can be placed. |
| Use mode | `UseMode` | Engine placement/action mode for the item. |

### WYSIWYG Editor Safety Rules

- Preserve comments, unknown properties, unknown attributes, nested action classes, and effect groups.
- Validate XML before saving, but only warn on unresolved references because another mod may provide the referenced item, block, material, buff, or asset.
- Normalize booleans and casing to match the existing file style when possible.
- Do not assume every numeric-looking value is a single number; 7DTD often uses ranges or comma-separated values such as durability ranges and RGB colors.
- Show XPath patch operations (`append`, `set`, `setattribute`, `remove`) as first-class operations, not just raw text.
- Back up `Config/items.xml` before automated rewrites.
