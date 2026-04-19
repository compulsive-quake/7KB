# 7 Days to Die Modding Knowledgebase

Shared knowledgebase for 7DTD mod development. Used by multiple projects (7nes, 7builder, PixelPaste).

## Modding Fundamentals

- [Mod Structure](Mod%20Structure.md) — Folder layout, ModInfo.xml, DLL loading, modlet system
- [XML Patching (XPath)](XML%20Patching%20(XPath).md) — How to patch vanilla XML configs; sounds system
- [Harmony Patching](Harmony%20Patching.md) — Harmony prefix/postfix patterns, common targets, reflection
- [Localization](Localization.md) — Adding translated text keys
- [Blocks](Blocks.md) — Custom block classes, XPath patches, block properties
- [Workstations](Workstations.md) — Workstation blocks, modules, tool slots, XUi window groups
- [Items](Items.md) — Custom item definitions, icons (ItemIconAtlas), dynamic item generation
- [Power Sources](Power%20Sources.md) — Custom power generators, TileEntityPowerSource, fuel management, power system C# internals
- [Entities](Entities.md) — Custom entity classes (NPCs, zombies, animals), AI tasks, drops
- [Networking - Connecting and Chat](Networking%20-%20Connecting%20and%20Chat.md) — ConnectionManager, sending chat via NetPackageChat, JSON escape gotcha in 7debug's /api/command
- [Asset Bundles](Asset%20Bundles.md) — Loading custom models, particles, sounds from .unity3d bundles
- [Paint & Textures](Paint%20%26%20Textures.md) — Block paint texture IDs and the paint system

## XUi (UI System)

- [XUi - Window System](XUi%20-%20Window%20System.md) — XML window definitions, elements, colors, styles, controls
- [XUi - Controllers (C#)](XUi%20-%20Controllers%20(C%23).md) — C# XUiController subclasses, bindings, events

## Prefab & Binary Formats

- [TTS File Format](TTS%20File%20Format.md) — TerraTerrain Storage binary format (voxel grid)
- [NIM File Format](NIM%20File%20Format.md) — Name Index Map binary format (block ID → name)
- [Block Data Structures](Block%20Data%20Structures.md) — Key data structures for prefab import
- [Block Definition Resolution](Block%20Definition%20Resolution.md) — How block IDs resolve to definitions
- [Prefab Import Pipeline](Prefab%20Import%20Pipeline.md) — Full flow from prefab selection to placed voxels

## Coordinate & Rotation Systems

- [Coordinate System](Coordinate%20System.md) — Prefab grid, transforms, TTS iteration order
- [Entity Positions](Entity%20Positions.md) — Live runtime world coordinates, origin reposition, sanity ranges, JSON serialization gotchas
- [Rotation System](Rotation%20System.md) — 24 discrete block orientations
- [Rotation Types](Rotation%20Types.md) — Per-block rotation constraints
- [Multi-Block Rotation](Multi-Block%20Rotation.md) — How rotation affects multi-cell blocks
- [X-Mirror Rotation](X-Mirror%20Rotation.md) — Rotation correction for mirrored prefabs

## Quick Reference

| Topic | Key File (vanilla) |
|---|---|
| All window/HUD definitions | `Data/Config/XUi/windows.xml` |
| Reusable UI component templates | `Data/Config/XUi/controls.xml` |
| Named color tokens & styles | `Data/Config/XUi/styles.xml` |
| XUi ruleset / window group registration | `Data/Config/XUi/xui.xml` |
| Block definitions | `Data/Config/blocks.xml` |
| Entity class definitions | `Data/Config/entityclasses.xml` |
| Item definitions | `Data/Config/items.xml` |
| Item modifier definitions | `Data/Config/item_modifiers.xml` |
| Quality/item background colors | `Data/Config/qualityinfo.xml` |
| UI display info (item stats shown in UI) | `Data/Config/ui_display.xml` |
| Buff/effect definitions | `Data/Config/buffs.xml` |
| Recipe definitions | `Data/Config/recipes.xml` |
| Sound definitions | `Data/Config/sounds.xml` |
| Loot table definitions | `Data/Config/loot.xml` |
| Loading screen tips | `Data/Config/loadingscreen.xml` |
