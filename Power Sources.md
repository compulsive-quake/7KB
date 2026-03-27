# Power Sources

Part of the [7DTD Modding Knowledgebase](README.md). Covers custom power source blocks, TileEntity hierarchy, fuel management, and the power system's C# internals.

---

## Overview

Power sources are blocks that generate electrical power for the wire system. Vanilla examples include the generator bank and solar bank. The game uses a class hierarchy:

```
Block
  └── BlockPowered              ← consumes power (requires wire)
       └── BlockPowerSource     ← generates power
```

```
TileEntity
  └── TileEntityPowered         ← any powered block
       └── TileEntityPowerSource ← power generators
```

```
PowerItem
  └── PowerSource               ← runtime power state
       └── PowerGenerator       ← generators with fuel
```

---

## Block Definition (XML)

Power source blocks use `<append xpath="/blocks">` (NOT `<config>` with `<append xpath="/blocks">` in a `<configs>` wrapper — either form works but be consistent):

```xml
<block name="myGenerator">
    <property name="Class" value="BlockMyGenerator, mymod"/>
    <property name="SlotItem" value="smallEngine"/>
    <property name="OutputPerFuel" value="11250"/>
    <property name="OutputPerStack" value="100"/>
    <property name="Shape" value="ModelEntity"/>
    <property name="Model" value="#@modfolder:Resources/mymodel.unity3d?myprefab.prefab"/>
    <property name="Material" value="Mmetal_medium"/>
    <property name="CreativeMode" value="Player"/>
    <property name="CustomIcon" value="myicon"/>
    <property name="HandleFace" value="Bottom"/>
    <property name="Collide" value="movement,melee,bullet,arrow,rocket"/>
    <property name="Path" value="solid"/>
    <property name="StabilitySupport" value="false"/>
</block>
```

### Key Properties

| Property | Description |
|---|---|
| `Class` | C# class. Use `"ClassName, AssemblyName"` for mod classes. Vanilla: `BlockPowerSource` |
| `SlotItem` | Item required in the generator slot (e.g., `smallEngine`) |
| `OutputPerFuel` | Power ticks per unit of fuel |
| `OutputPerStack` | Power output per fuel stack |
| `Shape` | `ModelEntity` for 3D model blocks |
| `Model` | `#@modfolder:Resources/bundle.unity3d?prefab.prefab` for custom models |

---

## Custom Block Class (C#)

Extend `BlockPowerSource` for a custom power generator:

```csharp
[Preserve]
public class BlockMyGenerator : BlockPowerSource
{
    public override TileEntityPowerSource CreateTileEntity(Chunk chunk)
    {
        if (slotItem == null)
            slotItem = ItemClass.GetItemClass(SlotItemName, false);

        var te = new TileEntityMyGenerator(chunk)
        {
            PowerItemType = PowerItemTypes.Generator // = 5
        };
        te.SlotItem = slotItem;
        return te;
    }

    public override string GetPowerSourceIcon()
    {
        return "electric_generator";
    }
}
```

---

## Custom TileEntity

Extend `TileEntityPowerSource`:

```csharp
[Preserve]
public class TileEntityMyGenerator : TileEntityPowerSource
{
    public TileEntityMyGenerator(Chunk chunk) : base(chunk) { }

    public override void OnSetLocalChunkPosition()
    {
        base.OnSetLocalChunkPosition();

        // Server-side: create power item if missing
        if (ConnectionManager.Instance.IsServer && PowerItem == null)
        {
            BlockValue bv = chunk.GetBlock(localChunkPos);
            PowerItem = CreatePowerItemForTileEntity((ushort)bv.type);

            if (PowerItem is PowerSource ps)
                ps.AddTileEntity(this);

            SetModified();
        }
    }

    public override void UpdateTick(World world)
    {
        base.UpdateTick(world);
        // Custom tick logic here
    }
}
```

---

## Power System Runtime Classes

### PowerSource / PowerGenerator

Key properties on `PowerSource`:

| Property | Type | Description |
|---|---|---|
| `CurrentPower` | `int` | Current power being generated |
| `MaxOutput` | `int` | Maximum power output (watts) |
| `MaxPower` | `int` | Maximum power capacity |
| `LastPowerUsed` | `int` | Power consumed by connected devices |
| `IsOn` | `bool` | Whether the source is turned on |
| `OutputPerStack` | `int` | Output per fuel stack |

Key properties on `PowerGenerator` (extends `PowerSource`):

| Property | Type | Description |
|---|---|---|
| `CurrentFuel` | `ushort` | Current fuel level |
| `MaxFuel` | `int` | Maximum fuel capacity |

### Checking running state from block meta

The running state is encoded in `BlockValue.meta`:
```csharp
bool isRunning = (blockValue.meta & 2) != 0;
```

### Power system hooks

| Method | Class | Purpose |
|---|---|---|
| `TickPowerGeneration()` | `PowerGenerator` | Called each tick to generate power |
| `RefreshPowerStats()` | `PowerSource` | Recalculates power stats |
| `HandleSendPower()` | `PowerSource` | Distributes power to children |
| `HandlePowerReceived(ref ushort)` | `PowerItem` | Called when a consumer receives power |
| `SendHasLocalChangesToRoot()` | `PowerItem` | Propagates state changes up the power tree |

---

## Custom UI for Power Sources

When a player interacts with a power source, the game opens `XUiC_PowerSourceWindowGroup`. To redirect to a custom window, use a Harmony prefix on `OnOpen`:

```csharp
[HarmonyPatch(typeof(XUiC_PowerSourceWindowGroup), "OnOpen")]
public class Patch_OpenCustomWindow
{
    static bool Prefix(XUiC_PowerSourceWindowGroup __instance)
    {
        // Get the tile entity via reflection (field is private)
        var teField = __instance.GetType().GetField("tileEntity",
            BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
        var te = teField?.GetValue(__instance) as TileEntityPowerSource;

        if (te == null || !(te is TileEntityMyGenerator))
            return true; // let vanilla handle it

        // Close vanilla window, open custom
        var wm = __instance.xui.playerUI.windowManager;
        wm.Close("powersource");
        wm.Open("MyGeneratorGroup", true);

        // Assign tile entity to custom controller
        var ctrl = __instance.xui.GetChildById("windowMyGenerator");
        // ... assign TE to controller ...

        return false; // skip vanilla OnOpen
    }
}
```

### Window definition for custom power source UI

Define in `Config/XUi/windows.xml`:
```xml
<configs>
  <append xpath="/windows">
    <window name="windowMyGenerator"
            width="520" height="320"
            cursor_area="true"
            controller="XUiC_MyGeneratorWindow, mymod">
      <!-- Layout with progress bars, buttons, labels -->
      <!-- Use {binding} text for dynamic values -->
      <!-- Use sprite type="filled" fill="0" for progress bars -->
    </window>
  </append>
</configs>
```

Register the window group in `Config/XUi/xui.xml`:
```xml
<config>
  <append xpath="/xui/ruleset">
    <window_group name="MyGeneratorGroup" close_compass_on_open="true">
      <window name="windowMyGenerator"/>
      <window name="windowNonPagingHeader"/>
    </window_group>
  </append>
</config>
```

> **Note:** Custom power source UIs do NOT use the workstation system. They define their own window groups and controllers.

---

## Filled Sprites (Progress Bars)

Use `type="filled"` sprites for progress bars. The `fill` attribute (0.0–1.0) controls how much is shown:

```xml
<sprite depth="5" name="sprFill" color="200,40,40,200"
        width="300" height="30" type="filled" fill="0"/>
```

In C#, update the fill via the `XUiV_Sprite`:
```csharp
XUiController ctrl = GetChildById("sprFill");
XUiV_Sprite spr = ctrl?.ViewComponent as XUiV_Sprite;
if (spr != null)
    spr.Fill = 0.75f; // 75% filled
```

---

## Fuel Management

### Fuel via PowerGenerator

`PowerGenerator.CurrentFuel` tracks coal/gas. Consume on a timer:

```csharp
gen.CurrentFuel = (ushort)Mathf.Max(0, gen.CurrentFuel - 1);
```

### Custom fuel (e.g., water)

The game doesn't natively support a second fuel type. Options:
1. **ConditionalWeakTable** — attach runtime data to the TileEntity (lost on restart unless serialized)
2. **TileEntity PersistentData** — some builds expose a `PersistentData` dictionary on TileEntity
3. **Patch read/write** — Harmony patches on `TileEntityX.read()` / `write()` to serialize custom data

Example using a marker byte in the save stream:
```csharp
// In write postfix: append marker + data
writer.Write((byte)90);       // marker byte
writer.Write((ushort)waterFuel);

// In read postfix: check for marker
if (HasMoreBytes(reader))
{
    byte marker = reader.ReadByte();
    if (marker == 90)
        waterFuel = reader.ReadUInt16();
    else
        RewindOneByte(reader); // not our data, put it back
}
```

> **Important:** Always check `HasMoreBytes` and use a unique marker byte to avoid reading past the stream or corrupting other data.

---

## Consuming Items from Player Inventory

To transfer items from the player's inventory to a generator (e.g., "Add Coal" button):

```csharp
EntityPlayerLocal player = xui.playerUI.entityPlayer;
ItemValue coalItem = ItemClass.GetItem("resourceCoal", false);
int available = player.bag.GetItemCount(coalItem);
int toAdd = Mathf.Min(available, 250);
if (toAdd > 0)
{
    player.bag.DecItem(coalItem, toAdd);
    gen.CurrentFuel = (ushort)Mathf.Min(gen.MaxFuel, gen.CurrentFuel + toAdd);
    te.SetModified();
    gen.SendHasLocalChangesToRoot();
}
```

---

## Block Animator Control

For blocks with animated models, find and control the `Animator`:

```csharp
BlockEntityData bed = chunkSync.GetBlockEntity(blockPos);
if (bed != null && bed.bHasTransform)
{
    Animator anim = bed.transform.GetComponentInChildren<Animator>(true);
    if (anim != null)
    {
        // Check parameter exists before setting
        anim.SetBool("Running", isRunning);
        anim.SetTrigger("Start");
    }
}
```

Always check if an animator parameter exists before setting it to avoid errors:
```csharp
bool HasParam(Animator anim, string name, AnimatorControllerParameterType type)
{
    foreach (var p in anim.parameters)
        if (p.name == name && p.type == type) return true;
    return false;
}
```

---

## Particle Effects

Spawn particle effects from asset bundles at runtime:

1. Load prefab from bundle: `AssetBundle.LoadFromFile(path)` then `bundle.LoadAsset<GameObject>(name)`
2. Instantiate as child of the block transform
3. Toggle emission on/off based on running state
4. Use `ParticleSystem.emission.enabled` and fade alpha for smooth transitions

---

## Drop Items on Block Destroy

When a custom block is destroyed, drop its contents:

```csharp
[HarmonyPatch(typeof(TileEntityPowerSource), "ReplacedBy")]
public static class Patch_DropOnDestroy
{
    static bool Prefix(TileEntityPowerSource __instance, ...)
    {
        if (!(__instance is TileEntityMyGenerator)) return true;

        List<ItemStack> drops = new List<ItemStack>();
        // Collect fuel, water, slot items...

        Vector3 dropPos = __instance.ToWorldCenterPos();
        dropPos.y += 0.9f;
        GameManager.Instance.DropContentInLootContainerServer(
            -1, "DroppedLootContainer", dropPos,
            drops.ToArray(), true, null);

        return false; // skip vanilla
    }
}
```

---

## Gotchas

> **`Class` attribute with assembly name** — For mod block classes, use `Class="BlockMyGen, myassembly"` format. Without the assembly name, the game only searches `Assembly-CSharp`.

> **`GUIWindowManager.Open` signature varies by game version** — Use reflection to find the correct overload. Common signatures: `Open(string, int, int, bool, bool)` and `Open(string, bool)`.

> **`TileEntity.SetModified()` is essential** — Always call after changing any TileEntity state. Without it, changes may not persist or sync to clients.

> **Dedicated server checks** — Guard visual-only code (particles, sounds, UI) with `if (GameManager.IsDedicatedServer) return;`

> **`[Preserve]` attribute** — Add to classes that are only referenced via XML/reflection to prevent Unity's managed code stripping from removing them.
