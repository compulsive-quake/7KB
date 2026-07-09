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

### Collision / "can be walked through" (C#)

`Block.BlockingType` is a bitmask; the per-channel booleans are derived from it:

| Property | Bit | Meaning |
|---|---|---|
| `IsCollideMovement` | `& 2` | Blocks entity/player movement. **False for grass, plants, and other pass-through decorations.** Use this to tell "is this block actually solid to walk into" — air-vs-non-air (`BlockValue.isair`) is not enough, since grass is a non-air block you can walk through. |
| `IsCollideSight` | `& 1` | Blocks line of sight |
| `IsCollideBullets` | `& 4` | Stops bullets |
| `IsCollideRockets` | `& 8` | Stops rockets/explosives |
| `IsCollideMelee` | `& 0x10` | Stops melee |
| `IsCollideArrows` | `& 0x20` | Stops arrows |

For face-aware solidity (partial shapes), use `Block.IsMovementBlocked(world, pos, blockValue, face)`.

### Exact-shape collision — don't approximate blocks as full cubes (C#)

There is **no public per-block collision AABB / shape-extent API**: no `BlockShape`
type is even resolvable in Assembly-CSharp (checked 2026-07; `Block` itself fails
plain reflection load due to unresolvable deps). Consequences for custom movers
(drones, projectiles with volume):

- Any voxel-grid overlap test (`world.GetBlock` + "does my sphere/box touch the
  1m cell") treats partial blocks — plates, bars, window frames, half-blocks,
  railings — as **full cubes** and over-blocks gaps that visually exist.
- For collision that respects real partial-block geometry, query Unity physics
  against the **baked chunk MeshColliders**: `Physics.BoxCastAll` / `Physics.OverlapBox`
  (positions offset by `-Origin.position`). They carry the exact shape geometry.
  Colliders only exist where chunks are *displayed* — a `ChunkObserver` keeps them
  baked around a remote object.
- `Voxel.Raycast(world, ray, dist, -538750997, 8, 0f)` (the bullet path) also
  respects partial-block shapes and is zero-width — a good backstop for unbaked
  chunks: it can under-block but never over-block.
- Size the query from the mesh: union `Renderer.bounds` at identity rotation for
  oriented-box half-extents. A bounding *sphere* inflates a flat object in every
  axis (FPV mod: a 0.65 m-wide flat quad collided like a 0.9 m sphere and
  couldn't thread 1 m openings).

### Crosshair "place where I'm looking" pattern (C#)

To set a custom object down at the crosshair like vanilla block placement
(FPV mod drone launch, `TryComputeGroundSpot`):

1. Cast the **camera** ray, not `EntityAlive.GetLookRay()` — for the local
   player that resolves to `EntityHuman`'s head-bone ray (off-pivot, no view
   bob), which drifts off the crosshair at steep pitch. Use
   `new Ray(player.cameraTransform.position + Origin.position, player.cameraTransform.forward)`
   (Voxel.Raycast wants un-shifted world space).
2. `Voxel.Raycast(world, ray, ~5f, -538750997, 8, 0f)`; hit at
   `Voxel.voxelRayHitInfo.hit.pos`. No hit within reach → no placement
   (that's the "can't place on air" rule; also hide any ghost preview).
3. Nudge the hit back out of the struck face (`hit - ray.direction * 0.1f`)
   so a wall hit probes the air in front of the block, then raycast **down**
   a bounded distance (~3 m) from slightly above that point to find the floor.
   No floor within the bound → refuse (aiming over a cliff edge is air).
4. Lift the result by the object's collision radius so it rests on, not in,
   the surface.

Validate once in the item action (`TryGet…` returning the denial tooltip
string) and pass the spot into the spawn/launch code instead of recomputing —
recomputing on a later frame can disagree with the preview the player saw.

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

### When the wheel shows vs. instant action (IMPORTANT)
`PlayerMoveController.Update` opens the radial on Activate **WasPressed** (first press, not a long hold) for any block whose `HasBlockActivationCommands(...)` returns true, then calls `XUiC_Radial.SetCurrentBlockData` → `SetCommonData`. That method counts only the **enabled** commands:
- **0 or 1 enabled command** → it calls `CallContextAction()` immediately and never draws the wheel (the single command just fires — e.g. opens the options window). A block that always returns exactly one enabled command will *never* show a radial.
- **2+ enabled commands** → the wheel is drawn; you keep Activate held, move to a slice, and release to pick.

So the "hold E to see a wheel" UX is an emergent property of having ≥2 *enabled* commands — there is no separate hold gesture to implement. Toggle `enabled` per-command to switch a block between instant-action and wheel.

### Vanilla turret behavior is free via `BlockRanged`
`Class=Ranged` (`BlockRanged : BlockPowered`) already returns exactly two commands — `options`/`tool` and `take`/`hand` — with land-claim-aware enablement, and is what the SMG/auto turret uses:
- `cmds[0] "options"` enabled when `_world.CanPlaceBlockAt(pos, localPlayer)` (true in your claim or unclaimed land).
- `cmds[1] "take"` enabled when `_world.IsMyLandProtectedBlock(pos, localPlayer)` **and** `TakeDelay > 0f`.

Net effect, no extra code: **inside your own land claim → 2 enabled → radial wheel (options + take); outside a claim → 1 enabled → options window opens instantly.** A mod block that just `extends BlockRanged` and delegates `GetBlockActivationCommands`/`OnBlockActivated` to `base` inherits this exactly (verified with RocketTurret). `TakeDelay` lives on `BlockPowered` (default `2f`, override via the `TakeDelay` block property), so "take" is enabled in-claim even if you never set it.

Note: code/`.dll` changes are restart-kind — a deployed turret refactor only changes behavior after the client is fully restarted, not on `xui reload`.

### Icon resolution
The game **prepends `ui_game_symbol_`** to the icon name. So `"hand"` resolves to `"ui_game_symbol_hand"` in the UIAtlas.

Common built-in icons: `hand`, `electric_switch`, `server_search`, `hammer`, `campfire`, `forge`, `workbench`, `map`, `skills`, `quest`, `lock`, `search`, `camera`, `drop_item`, `rotate_simple`, `paint_bucket`, `zombie`, `lightbulb`, `character`, `players`, `chat`, `all_blocks`, `copy_shape`, `x`.

Full list of `ui_game_symbol_*` icons available in the UIAtlas (as of V2.6 b14): `add`, `agility`, `all_blocks`, `allies`, `armor_iron`, `arrow_left`, `arrow_max`, `arrow_menu`, `arrow_right`, `assemble`, `backpack`, `block_damage`, `block_repair`, `book`, `brick`, `cement`, `chair`, `challenge`, `character`, `check`, `climb`, `clock`, `coin`, `compass`, `computer`, `cookware`, `crops`, `defense`, `destruction`, `diamond`, `door`, `drop`, `electric_max_power`, `electric_plugin`, `electric_power`, `electric_switch`, `fire`, `foliage`, `forge`, `fork`, `frames`, `gas`, `go_to_button`, `hammer`, `hand`, `ibeam`, `junk_turret`, `knife`, `lightbulb`, `lock`, `loot_sack`, `map`, `map_bed`, `map_campsite`, `map_cave`, `map_cursor`, `map_house`, `map_player_arrow`, `map_town`, `map_trader`, `map_waypoint_remove`, `map_waypoint_set`, `medical`, `mic`, `minibike`, `modded`, `noise`, `other`, `pack_mule`, `paint_brush`, `paint_bucket`, `pen`, `percent`, `perk`, `ping`, `player`, `players`, `quest`, `quest_limited`, `quest_remove`, `radiation`, `report`, `resource`, `rifle`, `science`, `search`, `seats`, `service`, `shape_ammo`, `shape_factory`, `shape_outdoor_props`, `shape_schematics`, `shirt`, `shopping_cart`, `sight`, `signs`, `skills`, `skull`, `sort`, `speed`, `stealth`, `store_all_down`, `store_all_up`, `store_similar_down`, `store_similar_up`, `subtract`, `swap`, `teleport`, `temperature`, `tool`, `tool_smithing`, `traps`, `trash`, `treasure`, `tree`, `trophy`, `unlock`, `vending`, `wardrobe`, `wind`, `windows`, `wood`, `wrench`, `x`, `zombie`.

### Localization
The radial menu looks up `blockcommand_{commandName}` in localization for built-in block activation commands. If the key is missing, the radial menu displays the raw key, e.g. `blockcommand_my_action`.

Add entries in `Config/Localization.txt` using the full vanilla localization CSV header/row width:
```csv
Key,File,Type,UsedInMainMenu,NoTranslate,english,Context / Alternate Text,german,spanish,french,italian,japanese,koreana,polish,brazilian,russian,turkish,schinese,tchinese
blockcommand_my_action,UI,Radial blockcommand,,,My Action,,My Action,My Action,My Action,My Action,My Action,My Action,My Action,My Action,My Action,My Action,My Action,My Action
```

Do not use a shortened six-column localization file for radial command keys. It can fail to load the key correctly, leaving the radial menu with raw text such as `blockcommand_sentryLightTurnOff`.

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
| `BlockRanged` | Powered SMG/shotgun-style turret with ammo-slot UI | `TileEntityPoweredRangedTrap` |
| `BlockPowerSource` | Generates power | `TileEntityPowerSource` |
| `BlockWorkstation` | Crafting stations | `TileEntityWorkstation` |
| `BlockForge` | Smelting stations | `TileEntityWorkstation` |
| `BlockCampfire` | Cooking stations | `TileEntityWorkstation` |

### Powered ranged trap ammo UI

Use `BlockRanged`/`TileEntityPoweredRangedTrap` when a custom placed trap should behave like the vanilla SMG or shotgun auto turret and open the normal `powerrangedtrap` window with ammo slots. A plain `Block` with custom activation commands can show a radial menu, but it will not get the vanilla "E to interact" prompt or ammo-slot window.

Minimum XML properties mirror vanilla `autoTurret`:

```xml
<property name="Class" value="BlockMyTurret, mymod"/>
<property name="RequiredPower" value="15"/>
<property name="AmmoItem" value="ammo9mmBulletBall+tags(ammo9mm)"/>
<property name="ShowTargeting" value="true"/>
```

In C#, inherit from `BlockRanged`, call base activation methods, and bind runtime behavior to `TileEntityPoweredRangedTrap.ItemSlots`/`DecrementAmmo(...)` rather than storing a parallel ammo count. If migrating an existing save from a custom `TileEntity`, replace stale tile entities with `CreateTileEntity(chunk) as TileEntityPoweredRangedTrap` and carry over old ammo before opening the window.

**Reading loaded ammo (per type + count).** `ItemSlots` is the member that gives you both: it's `ItemStack[]` (each non-empty slot has `.itemValue` and `.count`), so to tally rockets by type across turrets, iterate `ItemSlots`, skip `count <= 0`, group by `stack.itemValue.ItemClass.Name`, and label rows with `ItemClass.GetLocalizedItemName()`. Don't reach for the obvious-sounding members — verified against the assembly (RocketTurret, June 2026): `AmmoItems` is `ItemClass[]` (the accepted/loaded ammo *types*, no counts); `CurrentAmmoItem()` returns only the single active `ItemClass`; and `GetAmmoCount()`/`GetItems()`/`GetSlots()` exist elsewhere in Assembly-CSharp but are **NOT** on `TileEntityPoweredRangedTrap` (a raw string-grep finds the name globally and misleads — reflect the actual type with an `AssemblyResolve` pointing at `…/Managed` to confirm membership before coding against it).

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

> **`GameManager.ExplosionServer` can't be made block-safe by zeroing `Explosion.BlockDamage`/`RadiusBlocks`.** In `Explosion.AttackBlocks` the radius is floored to `0.01` (so `num2 = CeilToInt = 1`, the 3×3×3 epicenter loop always runs), and when the explosion is attributed to an `EntityAlive` instigator the per-block damage is computed as `BlockDamage * blockDamageScale + 0.5f` — i.e. a hard-coded **+0.5 minimum** at the epicenter regardless of your props. For most blocks the resulting rounded damage is 0, but **hardness-0 props (parked cars, some decorations) take the `else` branch `num8 = MaxDamage / landProtMod`, destroying them in one hit** → they downgrade to their burnt wreck `DowngradeBlock`. Passing `entityId = -1` makes block damage truly 0 (the `+0.5` is gated on a non-null instigator) but loses kill XP / quest events. To shred entities while *never* touching blocks AND keeping XP, skip `ExplosionServer` entirely and apply damage yourself: `Physics.OverlapSphere(pos - Origin.position, radius, -538480645)` → filter colliders whose tag starts with `E_BP_` → `GameUtils.GetHitRootTransform(tag, t).GetComponent<EntityAlive>()` → `entity.DamageEntity(new DamageSourceEntity(EnumDamageSource.External, EnumDamageTypes.Piercing, ownerId, dir, t.name, Vector3.zero, Vector2.zero){AttackingItem=item}, dmg, false)`, then `(owner as EntityPlayer)?.AddKillXP(entity)` on a fresh kill. (Airstrike mod uses exactly this.)

> **`world.SetBlock` and stability**: The 4th parameter of `world.SetBlock(clrIdx, pos, blockValue, updatePhysics, notify)` controls whether structural integrity (SI) is recalculated. If set to `false`, placed blocks will have a stability of 1 instead of their proper value. Always pass `true` for `updatePhysics` when placing blocks that need correct stability (e.g. building structures). Hand-placed blocks go through the game's normal placement path which always recalculates SI.

> **Powered-ranged-trap / turret `AmmoItem` accepts only ONE item unless you add a tag filter.** The ammo-slot filter for `BlockPoweredRangedTrap`-style turrets is driven by the block's `AmmoItem` property. A bare value like `AmmoItem="ammoRocketHE"` accepts *only* that exact item. To accept multiple ammo variants, use the vanilla `+tags(...)` syntax — the slot then accepts any item carrying one of those tags. Vanilla examples: SMG turret `ammo9mmBulletBall+tags(ammo9mm)`, shotgun turret `ammoShotgunShell+tags(ammoShells)`. Pick a tag the variants share but nothing else loadable does — e.g. rockets `ammoRocketHE`/`ammoRocketFrag` only share `ammo` (too broad) and `perkDemolitionsExpert` (also on grenades/explosive arrows), so add a custom tag (`ammoRocket`) to the base ammo item and filter on that. Because `ammoRocketFrag` `Extends` `ammoRocketHE` without redefining `Tags`, patching the tag onto HE propagates to Frag automatically.
