# Workstations

Part of the [7DTD Modding Knowledgebase](README.md). Covers Workstation block configuration — block properties, XUi window setup, modules, and tool slots.

---

## Overview

Workstations are interactive blocks that open a crafting/storage UI when the player presses E. The game has several workstation block classes (`Workstation`, `Forge`, `Campfire`) that all use `TileEntityWorkstation` and the `XUiC_WorkstationWindowGroup` controller.

---

## Block Definition

```xml
<block name="myWorkstation">
    <property name="Class" value="Workstation"/>
    <property class="Workstation">
        <property name="Modules" value="tools,output"/>
        <property name="CraftingAreaRecipes" value="myWorkstation"/>
        <property name="ToolNames" value="1"/>
    </property>
    <property name="WorkstationIcon" value="ui_game_symbol_wrench"/>
    <property name="Shape" value="New"/>
    <property name="Model" value="@:Shapes/Cube_A.fbx"/>
    <property name="OpenSound" value="UseActions/open_safe"/>
    <property name="CloseSound" value="UseActions/close_safe"/>
</block>
```

### Workstation Sub-Properties (`<property class="Workstation">`)

| Property | Required | Description |
|---|---|---|
| `Modules` | Yes | Comma-separated list of module types (see below) |
| `CraftingAreaRecipes` | Yes | Comma-separated recipe area names. Recipes with matching `craft_area` appear in the crafting list. Use a non-existent name (e.g., your block name) to have an empty crafting list. |
| `ToolNames` | For `tools` module | Comma-separated slot identifiers (e.g., `"1"` or `"1,2,3"`). These are slot IDs, not item names. |
| `InputMaterials` | For `material_input` module | Comma-separated material names (forge only) |

### Module Types

| Module | Purpose | Associated Window |
|---|---|---|
| `tools` | Tool/item slots (e.g., forge molds, anvil) | Custom tool window with `WorkstationToolGrid` controller |
| `output` | Output slot grid for crafted items | `windowOutput` |
| `fuel` | Fuel slot for burnable items | `windowFuel` |
| `input` | Generic input slots | Standard input window |
| `material_input` | Smeltable material input (forge) | `windowForgeInput` |

> **Important:** `CraftingAreaRecipes` appears to be required for `BlockWorkstation` to function. Without it, the E interaction may fail silently.

---

## Vanilla Workstation Examples

### Workbench (simplest)
```xml
<property name="Class" value="Workstation"/>
<property class="Workstation">
    <property name="Modules" value="output"/>
    <property name="CraftingAreaRecipes" value="player,workbench"/>
</property>
```

### Forge (tools + fuel + materials)
```xml
<property name="Class" value="Forge"/>
<property class="Workstation">
    <property name="CraftingAreaRecipes" value="forge"/>
    <property name="Modules" value="tools,output,fuel,material_input"/>
    <property name="InputMaterials" value="iron,brass,lead,glass,stone,clay"/>
    <property name="ToolNames" value="1,2,3"/>
</property>
```

### Campfire
```xml
<property name="Class" value="Campfire"/>
<property class="Workstation">
    <property name="Modules" value="tools,output,fuel,input"/>
</property>
```

---

## XUi Window Group

Every workstation needs a window group in `Config/XUi/xui.xml` named `workstation_{blockName}`:

```xml
<configs>
    <append xpath="/xui/ruleset[@name='default']">
        <window_group name="workstation_myWorkstation"
                      controller="XUiC_WorkstationWindowGroup"
                      open_backpack_on_open="true"
                      close_compass_on_open="true">
            <window name="windowCraftingList"/>
            <window name="craftingInfoPanel"/>
            <window name="windowCraftingQueue"/>
            <window name="windowMyTools"/>    <!-- custom tool window -->
            <window name="windowOutput"/>
            <window name="windowNonPagingHeader"/>
        </window_group>
    </append>
</configs>
```

> **Critical:** The xpath must be `/xui/ruleset[@name='default']`, NOT `/xui`. Window groups live inside the ruleset element. Using the wrong xpath causes `Window 'workstation_X' not found in XUI`.

### Required Windows

The `XUiC_WorkstationWindowGroup` controller expects these standard windows to exist in the group:

| Window | Purpose |
|---|---|
| `windowCraftingList` | Recipe list (vanilla, **required** — see warning below) |
| `craftingInfoPanel` | Recipe detail panel (vanilla, replaceable with custom window) |
| `windowCraftingQueue` | Active crafting queue (vanilla, **required** — see warning below) |
| `windowOutput` | Output grid (vanilla, include if `output` module used) |
| `windowFuel` | Fuel slot (vanilla, include if `fuel` module used) |
| `windowNonPagingHeader` | Header bar (vanilla, always include) |

> **WARNING: `windowCraftingList` and `windowCraftingQueue` cannot be removed.** `XUiC_WorkstationWindowGroup` inherits from `XUiC_CraftingWindowGroup`, whose `Init()` expects these windows and throws `NullReferenceException` if they are missing. The window group fails to initialize and becomes unavailable, producing `Window 'workstation_X' not found in XUI!` at runtime.

> **WARNING: Empty recipe lists crash.** If `CraftingAreaRecipes` names an area with no matching recipes, `XUiC_RecipeList.ClearSelection()` throws `NullReferenceException` during both `OnOpen` and `OnClose`. The recipe list's `set_SelectedEntry` property assumes internal state is initialized, which it isn't when there are zero recipe entries. **Workaround:** Add Harmony Finalizer patches on `XUiC_RecipeList.OnOpen` and `XUiC_RecipeList.OnClose` to suppress `NullReferenceException`:
>
> ```csharp
> [HarmonyPatch(typeof(XUiC_RecipeList), "OnOpen")]
> public class RecipeListOnOpenPatch
> {
>     static Exception Finalizer(Exception __exception)
>     {
>         if (__exception is NullReferenceException) return null;
>         return __exception;
>     }
> }
> ```
> Apply the same pattern for `OnClose`. See [Harmony Patching — Finalizer](Harmony%20Patching.md#finalizer-runs-regardless-of-exceptions).

You **can** replace `craftingInfoPanel` with a custom window. The controller does not require it by name.

### Replacing Default Panels with Custom Windows

For workstations that don't use crafting (e.g., a cartridge slot or single-purpose UI), the default `craftingInfoPanel` shows a useless "Inspect" help screen. Replace it with a custom static info window:

1. Define a custom window in `Config/XUi/windows.xml`:

```xml
<window name="windowMyInfo" width="530" height="420" panel="Center" cursor_area="true">
    <panel style="header.panel">
        <sprite style="header.icon" sprite="ui_game_symbol_wrench"/>
        <label style="header.name" text="My Custom Title" text_key="xuiMyTitle"/>
    </panel>
    <rect depth="0" pos="0,-46" height="374">
        <sprite depth="-2" pos="0,0" height="374" width="530"
                color="128,128,128,255" sprite="menu_empty" type="sliced"/>
        <label pos="16,-16" width="498" height="342" depth="1"
               text="Line one\nLine two\nLine three"
               font_size="24" color="220,220,220,255" justify="left" overflow="ClampContent"/>
    </rect>
</window>
```

2. Reference it in the window group instead of `craftingInfoPanel`:

```xml
<window_group name="workstation_myBlock" controller="XUiC_WorkstationWindowGroup"
              open_backpack_on_open="true" close_compass_on_open="true">
    <window name="windowMyInfo"/>        <!-- replaces craftingInfoPanel -->
    <window name="windowMyToolSlot"/>
    <window name="windowNonPagingHeader"/>
</window_group>
```

Key points:
- Use `panel="Center"` to position where `craftingInfoPanel` normally sits
- `\n` in label `text` attributes produces line breaks
- Add a `text_key` for localization, with the display text in `Config/localization.txt`
- The `sprite` with a grey `color` value and `type="sliced"` creates the standard dark panel background. Use direct `R,G,B,A` values (e.g., `"128,128,128,255"`) — bracket-style color references like `[medGrey]` are **not** valid XUi global style keys and produce `Global style key 'X' not found!` errors.

---

## Custom Tool Slot Window

Define a custom window in `Config/XUi/windows.xml` for the tool slot:

```xml
<configs>
    <append xpath="/windows">
        <window name="windowMyTools" width="228" height="121" panel="Right" cursor_area="true">
            <panel style="header.panel">
                <sprite style="header.icon" sprite="ui_game_symbol_wrench"/>
                <label style="header.name.shrink" text="TOOLS" text_key="xuiMyToolsHeader"/>
            </panel>
            <rect name="content" depth="0" pos="0,-46" height="75" disablefallthrough="true">
                <grid name="inventory" rows="1" cols="1" pos="3,-3"
                      cell_width="75" cell_height="75"
                      controller="WorkstationToolGrid"
                      repeat_content="true"
                      required_tools_only="false">
                    <item_stack controller="RequiredItemStack" name="0"/>
                </grid>
            </rect>
        </window>
    </append>
</configs>
```

> **Critical:** The `item_stack` elements inside a `WorkstationToolGrid` **must** have `controller="RequiredItemStack"`. Without it, `XUiC_WorkstationToolGrid.Init()` throws `InvalidCastException` ("Specified cast is not valid") because it casts children to `XUiC_RequiredItemStack`.

### Grid Attributes

| Attribute | Description |
|---|---|
| `controller="WorkstationToolGrid"` | Required — links to the workstation's tool module |
| `rows` / `cols` | Must match the number of tool slots (`ToolNames` count) |
| `required_tools="item1,item2"` | Restrict slots to specific items (like forge molds) |
| `required_tools_only="true"` | Only allow items listed in `required_tools` |
| `required_tools_only="false"` | Allow any item in the slot |

### Vanilla Tool Window (Forge)

The forge restricts each slot to a specific tool item:

```xml
<grid name="inventory" rows="1" cols="3" pos="3,-3" cell_width="75" cell_height="75"
      controller="WorkstationToolGrid" repeat_content="true"
      required_tools="toolBellows,toolAnvil,toolForgeCrucible" required_tools_only="true">
    <item_stack controller="RequiredItemStack" name="0"/>
</grid>
```

Note the `controller="RequiredItemStack"` on `item_stack` — this shows ghost icons for required tools.

---

## Block Classes

| Class Value | C# Class | Notes |
|---|---|---|
| `Workstation` | `BlockWorkstation` | Base workstation (workbench) |
| `Forge` | `BlockForge` | Extends workstation with smelting |
| `Campfire` | `BlockCampfire` | Extends workstation with cooking |

All use `TileEntityWorkstation` for storage and `XUiC_WorkstationWindowGroup` for UI.

> **T_Block tag required** — If your workstation uses a custom 3D model (`Shape="ModelEntity"`), the prefab's root GameObject **must** have the `T_Block` tag and a `BoxCollider`. Without this, the block renders but the E-press interaction prompt never appears — the player cannot open the workstation UI. See [Asset Bundles — Prefab structure requirements](Asset%20Bundles.md#prefab-structure-requirements).

> **Workstations vs Power Sources** — Workstations (`BlockWorkstation`) and power sources (`BlockPowerSource`) are completely separate systems. Workstations use `TileEntityWorkstation` + `XUiC_WorkstationWindowGroup`, while power sources use `TileEntityPowerSource` + `XUiC_PowerSourceWindowGroup`. If your block needs to both craft and generate power, you need Harmony patches to bridge the two systems. See [Power Sources](Power%20Sources.md).

---

## When NOT to Use a Workstation

If your block only needs item storage slots (no crafting), **do not use `Class="Workstation"`**. The workstation system brings in the full crafting UI (`XUiC_CraftingWindowGroup`, `XUiC_RecipeList`, etc.) which crashes when there are no recipes (see warnings in [Required Windows](#required-windows)).

Use `Class="SecureLoot"` instead. It provides:
- Persistent item storage (survives block unload/reload)
- Simple container UI (no crafting, no recipe list)
- `TileEntitySecureLootContainer` (extends `TileEntityLootContainer`) with a public `items` array

### SecureLoot Block Definition

```xml
<block name="myStorage">
    <property name="Class" value="SecureLoot"/>
    <property name="LootList" value="myStorageContainer"/>
    <property name="Shape" value="ModelEntity"/>
    <property name="Model" value="#@modfolder:Resources/mymodel.unity3d?..."/>
    <!-- other visual/physics properties -->
</block>
```

With a loot container in `loot.xml`:

```xml
<lootcontainer name="myStorageContainer" count="0" size="1,1"
               sound_open="UseActions/open_safe" sound_close="UseActions/close_safe"
               open_time="0" loot_quality_template="qualBaseTemplate">
</lootcontainer>
```

- `count="0"` — starts empty (no auto-generated loot)
- `size="columns,rows"` — e.g., `"1,1"` for 1 slot, `"3,2"` for 6 slots
- `open_time="0"` — instant open (no loot timer)

### Reading Items from SecureLoot in Code

```csharp
var te = world.GetTileEntity(clrIdx, pos) as TileEntityLootContainer;
if (te != null && te.items != null && te.items.Length > 0 && !te.items[0].IsEmpty())
{
    var itemClass = te.items[0].itemValue.ItemClass;
    string name = itemClass.GetItemName();
}
```

SecureLoot opens the default `looting` / `secureloot` window group — no custom XUi window group is needed.

> **Do not use `Extends` with vanilla block names** — Vanilla block names (e.g., `cntSecureStorageChest`, `cntStorageChest`) change between game versions. If the name doesn't exist, the block fails to parse and `blocks.xml` loading fails with `Could not find Extends block X`. Instead, define the block from scratch with explicit `Class` and properties. This avoids version-specific dependencies.
