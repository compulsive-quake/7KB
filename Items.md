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
