# Blocks

Part of the [7DTD Modding Knowledgebase](README.md). Covers custom block classes, block properties, and block activation.

---

## Registering a Custom Block

In `Config/blocks.xml`, add an XPath patch:

```xml
<configs>
    <append xpath="/blocks">
        <block name="MyBlock">
            <property name="Class" value="MyBlock" />
            <property name="Material" value="Mconcrete" />
            <property name="Shape" value="Cube" />
            <property name="Texture" value="14" />
            <property name="MaxDamage" value="1000" />
            <property name="IsTerrainDecoration" value="false" />
        </block>
    </append>
</configs>
```

The `Class` property value maps to the C# class `Block<Value>` — e.g. `Class="MyBlock"` loads `BlockMyBlock`.

---

## Custom Block C# Class

```csharp
public class BlockMyBlock : Block
{
    public override bool OnBlockActivated(
        string _interaction,
        WorldBase _world,
        int _cIdx,
        Vector3i _blockPos,
        BlockValue _blockValue,
        EntityAlive _player)
    {
        // Called when player presses E on the block
        EntityPlayerLocal localPlayer = _player as EntityPlayerLocal;
        if (localPlayer == null) return false;

        // Store anchor position for later use
        MyPlacer.AnchorPosition = _blockPos;

        // Open a UI window
        localPlayer.PlayerUI.windowManager.Open("myWindowGroup", true);
        return true;
    }
}
```

---

## Block Properties Reference

| Property | Example value | Notes |
|---|---|---|
| `Class` | `"MyBlock"` | C# class suffix (game prepends `Block`) |
| `Material` | `"Mconcrete"`, `"Msteel"` | Determines sound, resistance |
| `Shape` | `"Cube"`, `"ModelEntity"`, `"New"` | Block shape (see Shape Types below) |
| `Texture` | `"14"` | Default texture ID |
| `MaxDamage` | `"1000"` | Hit points |
| `IsTerrainDecoration` | `"false"` | Whether it floats on terrain |
| `ActivationText` | `"Open"` | Prompt shown when looking at block |

---

## Shape Types

The `Shape` property controls how the block is rendered:

| Shape | Usage | Notes |
|---|---|---|
| `ModelEntity` | Custom 3D models (prefabs from asset bundles) | **Use this for all custom model blocks.** Used by all vanilla workstations, power sources, and other blocks with custom 3D models. Requires `Model` property pointing to a prefab. |
| `New` | Standard voxel-based block shapes | For blocks using the game's built-in shape system |
| `Cube` | Simple cube block | Basic block shape |
| `Ext3dModel` | **DO NOT USE** | Despite existing in the code, this shape type is **not used by any vanilla block** and causes `InvalidCastException` in `BlockShapeExt3dModel.createVertices()` with custom prefabs. Always use `ModelEntity` instead. |

### Model property format

For blocks with `Shape="ModelEntity"`:
```xml
<!-- Vanilla model (from game's built-in prefabs) -->
<property name="Model" value="@:Entities/Crafting/woodWorkBenchPrefab.prefab"/>

<!-- Mod asset bundle model -->
<property name="Model" value="#@modfolder:Resources/mybundle.unity3d?Assets/MyPrefab.prefab"/>
```

---

## Block Activation Commands (Radial Menu)

Blocks can define a radial menu with multiple commands using `BlockActivationCommand` in C#:

```csharp
public override BlockActivationCommand[] GetBlockActivationCommands(...)
{
    return new BlockActivationCommand[]
    {
        new BlockActivationCommand("my_action", "electric_switch", true),
        new BlockActivationCommand("take", "hand", true)
    };
}
```

**Constructor**: `BlockActivationCommand(commandName, iconName, enabled)`

### Icon resolution
The game **prepends `ui_game_symbol_`** to the icon name. So `"hand"` resolves to `"ui_game_symbol_hand"` in the UIAtlas.

Common built-in icons: `hand`, `electric_switch`, `server_search`, `hammer`, `campfire`, `forge`, `workbench`, `map`, `skills`, `quest`, `lock`, `search`, `camera`, `drop_item`, `rotate_simple`, `paint_bucket`, `zombie`, `lightbulb`, `character`, `players`, `chat`, `all_blocks`, `copy_shape`, `x`.

Full list of `ui_game_symbol_*` icons available in the UIAtlas (as of V2.6 b14): `add`, `agility`, `all_blocks`, `allies`, `armor_iron`, `arrow_left`, `arrow_max`, `arrow_menu`, `arrow_right`, `assemble`, `backpack`, `block_damage`, `block_repair`, `book`, `brick`, `cement`, `chair`, `challenge`, `character`, `check`, `climb`, `clock`, `coin`, `compass`, `computer`, `cookware`, `crops`, `defense`, `destruction`, `diamond`, `door`, `drop`, `electric_max_power`, `electric_plugin`, `electric_power`, `electric_switch`, `fire`, `foliage`, `forge`, `fork`, `frames`, `gas`, `go_to_button`, `hammer`, `hand`, `ibeam`, `junk_turret`, `knife`, `lightbulb`, `lock`, `loot_sack`, `map`, `map_bed`, `map_campsite`, `map_cave`, `map_cursor`, `map_house`, `map_player_arrow`, `map_town`, `map_trader`, `map_waypoint_remove`, `map_waypoint_set`, `medical`, `mic`, `minibike`, `modded`, `noise`, `other`, `pack_mule`, `paint_brush`, `paint_bucket`, `pen`, `percent`, `perk`, `ping`, `player`, `players`, `quest`, `quest_limited`, `quest_remove`, `radiation`, `report`, `resource`, `rifle`, `science`, `search`, `seats`, `service`, `shape_ammo`, `shape_factory`, `shape_outdoor_props`, `shape_schematics`, `shirt`, `shopping_cart`, `sight`, `signs`, `skills`, `skull`, `sort`, `speed`, `stealth`, `store_all_down`, `store_all_up`, `store_similar_down`, `store_similar_up`, `subtract`, `swap`, `teleport`, `temperature`, `tool`, `tool_smithing`, `traps`, `trash`, `treasure`, `tree`, `trophy`, `unlock`, `vending`, `wardrobe`, `wind`, `windows`, `wood`, `wrench`, `x`, `zombie`.

### Localization
The radial menu looks up `blockcommand_{commandName}` in localization. Add entries in `Config/localization.txt`:
```
blockcommand_my_action,blocks,Block,FALSE,FALSE,My Action
```

### Custom icons

#### Item / inventory icons (toolbelt, backpack, creative menu)
Place a PNG in `UIAtlases/ItemIconAtlas/` — the game auto-discovers it by filename. Set the block's `CustomIcon` property to match the filename (without extension):
```xml
<property name="CustomIcon" value="myBlock"/>
```
with the icon at `UIAtlases/ItemIconAtlas/myBlock.png`. No C# code needed.

#### Radial menu icons
The `UIAtlases/UIAtlas/` folder does **not** work for radial menu icons — the game's UIAtlas `MultiSourceAtlasManager` doesn't pick them up automatically. To add custom icons, inject them at runtime via C# by:

1. Loading the PNG as a `Texture2D`
2. Creating a `UIAtlas` component with a `UISpriteData` entry (sprite name must include the `ui_game_symbol_` prefix)
3. Calling `MultiSourceAtlasManager.AddAtlas(atlas, true)` on the "UIAtlas" manager
4. Best done in a Harmony postfix on `XUi.Init()`

Requires references to `NGUI.dll` and `UnityEngine.ImageConversionModule.dll`.

### HasBlockActivationCommands

> **Gotcha:** You **must** override `HasBlockActivationCommands` to return `true`, or the game will never show the radial menu — even if `GetBlockActivationCommands` returns valid commands. The base `Block` class returns `false` by default.

```csharp
public override bool HasBlockActivationCommands(WorldBase _world, BlockValue _blockValue,
    int _clrIdx, Vector3i _blockPos, EntityAlive _entityFocusing)
{
    return true;
}
```

### Command handling
Override `OnBlockActivated(string _commandName, ...)` and switch on `_commandName` to handle each command. Return `true` to consume.

---

## Block Activation Interactions

The `_interaction` string in `OnBlockActivated` corresponds to the interaction type. Common values:
- `"E"` — default activate (press E)

Return `true` to consume the activation event (prevents default behavior).

---

## Activation Text Action Descriptors

When returning text from `GetActivationText`, you can embed keybind placeholders that the game resolves to the player's current key binding. The format is:

```
[action:actionSetName:ActionName]
```

The game expects a **colon** separator between the action set name and the action name. Using `[action:activate]` (no set name) causes:
```
WRN [XUi] Could not parse action descriptor in label text, no separator between action set name and action found
```

**Correct format** — use both `local` and `permanent` action sets together:
```csharp
__result = "Press [action:local:Activate][action:permanent:Activate] to use My Block";
```

This renders the correct keybind for both keyboard/mouse (local) and gamepad (permanent) input.

> **Gotcha:** The action name is case-sensitive. Use `Activate` (capital A), not `activate`.

---

## Placing Blocks from C#

```csharp
World world = GameManager.Instance.World;

// Get a block value by name
BlockValue bv = Block.GetBlockValue("steelShapes");

// Place the block
world.SetBlock(0, position, bv, false, false);

// Apply paint texture to a face
GameManager.Instance.SetBlockTextureServer(
    position,
    BlockFace.North,
    (byte)textureId,
    0  // player entity ID or 0
);
```

`BlockFace` values: `Top`, `Bottom`, `North`, `South`, `East`, `West`

---

## Block Class Hierarchy

The game has several block base classes for different purposes:

| Base Class | Purpose | TileEntity |
|---|---|---|
| `Block` | Basic static block | None |
| `BlockPowered` | Consumes power (wire target) | `TileEntityPowered` |
| `BlockPowerSource` | Generates power | `TileEntityPowerSource` |
| `BlockWorkstation` | Crafting stations | `TileEntityWorkstation` |
| `BlockForge` | Smelting stations | `TileEntityWorkstation` |
| `BlockCampfire` | Cooking stations | `TileEntityWorkstation` |

### Custom class in mod DLL

For mod block classes, use the `"ClassName, AssemblyName"` format in the `Class` property:
```xml
<property name="Class" value="BlockMyGenerator, mymod"/>
```
Without the assembly name, the game only searches `Assembly-CSharp`.

See also: [Power Sources](Power%20Sources.md) for custom power generator blocks, [Workstations](Workstations.md) for crafting stations.

---

## Gotchas

> **Block name in `Class`**: If `Class="PixelPaste"`, the game looks for a C# class named `BlockPixelPaste`. The `Block` prefix is prepended automatically.

> **`Block.GetBlockValue` returns `BlockValue.Air`** if the block name is not found. Always check `.isair` before using the result.

> **Vanilla block names**: There is no `steelBlock` or `concreteBlock` in vanilla 7DTD. Shape blocks may require a `:variant` suffix (e.g. `steelShapes:cube`) to resolve via `Block.GetBlockValue`. Plain `steelShapes` returns air in V 2.5 — always try the `:cube` variant first. Other notable blocks: `steelMaster`, `concreteMaster`, `concreteNoUpgradeMaster`.

> **`Shape="Ext3dModel"` crashes the game** — Despite existing in the code as `BlockShapeExt3dModel`, this shape type is not used by any vanilla block and throws `InvalidCastException` in `createVertices()` when loading custom prefabs. This error crashes the entire `blocks.xml` parser, which cascades into failures for items.xml, recipes.xml, loot.xml, quests.xml, traders.xml, and biomes.xml (since they all depend on blocks being loaded). **Always use `Shape="ModelEntity"` for custom 3D model blocks.**

> **`world.SetBlock` and stability**: The 4th parameter of `world.SetBlock(clrIdx, pos, blockValue, updatePhysics, notify)` controls whether structural integrity (SI) is recalculated. If set to `false`, placed blocks will have a stability of 1 instead of their proper value. Always pass `true` for `updatePhysics` when placing blocks that need correct stability (e.g. building structures). Hand-placed blocks go through the game's normal placement path which always recalculates SI.
