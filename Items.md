# Items

Part of the [7DTD Modding Knowledgebase](README.md). Covers custom item definitions, icons, and properties.

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

> **Strongly prefer authoring the prefab correctly over runtime transform adjustment.** The cleanest held item needs **zero** runtime transform code: bake the right pivot/scale into the prefab, put it on the `HoldingItem` layer (see below), and pick a `HoldType` in `items.xml`. Runtime "hand offset tuner" code that pokes the live held-item transform every frame is a recurring source of pain and was ultimately ripped out of RocketTurret:
> - Poking the held **root** with a cache to recover the game's base can latch onto a **world** position (the game doesn't reliably re-seat the root every frame for every setup) — the model then freezes in space while the player walks away (the laser/aim keeps tracking off the head vector, masking the cause).
> - Even offsetting the model's **children** instead is risky: if the engine reuses a shared held-item holder GameObject across items, your modifications can bleed into **other mods'** held items (their model shows up scaled/displaced/invisible). A user reported exactly this — "other apps' held items stopped showing when this mod is installed."
>
> If you genuinely need an offset, do it in the prefab, not at runtime. If you must do it at runtime, scope it so it only ever runs while *your* item is held and only touches *your* freshly-instantiated model, and treat any interaction with the shared held-item transform as suspect.

> **"I'm holding it but the hand looks empty" gotcha:** First decide whether it's invisible in *both* views or *only first person* — they have different causes.
>
> **Invisible in first person but VISIBLE in third person → wrong layer (most common for custom asset-bundle meshes).** 7DTD has a dedicated **`HoldingItem`** layer (confirmed in the game's layer table, alongside `Shadow`, `RenderInTexture`, `NGUI`). Vanilla held items live on this layer permanently; **both** the main camera (third person) and the first-person weapon camera (which draws the held item on top so it never clips through walls) render it. A custom bundle mesh comes in on its baked-in `Default` layer, which only the main camera draws → shows in third person, invisible in first. **The game does NOT relayer custom-attached models** (the tell: in third person the custom item is still sitting on `Default`, not `HoldingItem` — if the game relayered it, it wouldn't be on `Default`). So set the layer yourself the way the game authors its own items: put the whole held hierarchy on `HoldingItem`, **once**, when the model is instantiated — `int l = LayerMask.NameToLayer("HoldingItem"); foreach (Transform t in held.GetComponentsInChildren<Transform>(true)) t.gameObject.layer = l;`. Resolve by name (don't hard-code the index). No per-frame work and no first/third-person detection are needed — that was a wrong turn; propagating the *root's* layer each frame does nothing because the root stays `Default`. Trigger the one-time set off "held transform changed" in the `LateUpdate` hand tuner (a model refresh re-instantiates the transform, so this re-applies automatically). Ideally bake the layer into the prefab at build time instead, but runtime-by-name avoids depending on the Unity project's layer indices matching the game's.
>
> **Invisible in BOTH views → placement/pivot.** (1) The FBX geometry is offset from its own pivot, and that offset is *multiplied* by any visible-size normalization scale baked into the prefab (a 0.5m model auto-scaled ~3.7x turns a small modeling offset into a meters-large displacement) — re-center the mesh so its renderer-bounds center sits on the wrapper origin during the build. (2) It needs per-item offset tuning; if the mod ships a runtime hand-offset/HoldType tuner, confirm its `Register()` is actually **called** from `InitMod` (a defined-but-never-invoked registration leaves it stuck at the invisible default with no way to adjust). Verify the laser/muzzle child via `invData.model` to prove the mesh instantiated even when nothing is visible.

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
