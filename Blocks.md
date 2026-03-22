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
| `Shape` | `"Cube"` | Block shape |
| `Texture` | `"14"` | Default texture ID |
| `MaxDamage` | `"1000"` | Hit points |
| `IsTerrainDecoration` | `"false"` | Whether it floats on terrain |
| `ActivationText` | `"Open"` | Prompt shown when looking at block |

---

## Block Activation Interactions

The `_interaction` string in `OnBlockActivated` corresponds to the interaction type. Common values:
- `"E"` — default activate (press E)

Return `true` to consume the activation event (prevents default behavior).

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

## Gotchas

> **Block name in `Class`**: If `Class="PixelPaste"`, the game looks for a C# class named `BlockPixelPaste`. The `Block` prefix is prepended automatically.

> **`Block.GetBlockValue` returns `BlockValue.Air`** if the block name is not found. Always check `.isair` before using the result.

> **Vanilla block names**: There is no `steelBlock` or `concreteBlock` in vanilla 7DTD. Shape blocks may require a `:variant` suffix (e.g. `steelShapes:cube`) to resolve via `Block.GetBlockValue`. Plain `steelShapes` returns air in V 2.5 — always try the `:cube` variant first. Other notable blocks: `steelMaster`, `concreteMaster`, `concreteNoUpgradeMaster`.

> **`world.SetBlock` and stability**: The 4th parameter of `world.SetBlock(clrIdx, pos, blockValue, updatePhysics, notify)` controls whether structural integrity (SI) is recalculated. If set to `false`, placed blocks will have a stability of 1 instead of their proper value. Always pass `true` for `updatePhysics` when placing blocks that need correct stability (e.g. building structures). Hand-placed blocks go through the game's normal placement path which always recalculates SI.
