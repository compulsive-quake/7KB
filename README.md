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
- [Vehicles](Vehicles.md) — Custom vehicles (drivable mounts, hover bikes), `vehicles.xml` schema, mount/dismount pipeline, seat poses, camera switch, scripted-locomotion gotchas
- [Networking - Connecting and Chat](Networking%20-%20Connecting%20and%20Chat.md) — ConnectionManager, sending chat via NetPackageChat, JSON escape gotcha in 7debug's /api/command
- [Asset Bundles](Asset%20Bundles.md) — Loading custom models, particles, sounds from .unity3d bundles
- [Runtime Procedural VFX](Runtime%20Procedural%20VFX.md) — Runtime LineRenderer/particle effects with no bundle; the `HideAndDontSave` texture/material gotcha, procedural lightning, world→scene space
- [Player Feedback Sounds](Player%20Feedback%20Sounds.md) — Stock UI "denied" buzzers (e.g. `missingitemtorepair`) and how to play them via `Audio.Manager.PlayInsidePlayerHead`
- [Camera View Switching (First/Third Person)](Camera%20View%20Switching%20(First-Third%20Person).md) — `SetFirstPersonView`/`SetCameraAttachedToPlayer` internals; `playerCamera.transform` IS `cameraTransform`; the cross-mod bug where a temporary view switch leaves the camera + held-item transforms parked and breaks other mods' laser/aim (cone beam, targets own head), and the multi-frame re-assert fix
- [Paint & Textures](Paint%20%26%20Textures.md) — Block paint texture IDs and the paint system

## XUi (UI System)

- [XUi - Window System](XUi%20-%20Window%20System.md) — XML window definitions, elements, colors, styles, controls
- [XUi - Controllers (C#)](XUi%20-%20Controllers%20(C%23).md) — C# XUiController subclasses, bindings, events
- [HUD Safe Zones](HUD%20Safe%20Zones.md) — Top DAY/TIME strip + bottom message/toolbelt strip must not be overlapped by IMGUI or non-HUD-hiding windows
- [IMGUI Tracker Stack](IMGUI%20Tracker%20Stack.md) — Right-side stacked status cards for tracked entities (HP bar + status text + fade in/out). Also covers interactive IMGUI tool overlays (alignment / tuning sliders): always include a typeable text field next to step buttons, expose `static IsOpen` so other input listeners can short-circuit (`Event.current.Use()` doesn't block `Input.GetMouseButtonDown`).
- [zPhone External Apps](zPhone%20External%20Apps.md) — External app manifests, JSON page controls, action bridge registration, and held-item tuning patterns.

## Prefab & Binary Formats

- [TTS File Format](TTS%20File%20Format.md) — TerraTerrain Storage binary format (voxel grid)
- [NIM File Format](NIM%20File%20Format.md) — Name Index Map binary format (block ID → name)
- [Pregen World Format](Pregen%20World%20Format.md) — `Data/Worlds/` folder layout; `main.ttw` header gotcha
- [Extracting WEM Audio (Wwise)](Extracting%20WEM%20Audio%20(Wwise).md) — Pulling playable WAVs out of Helldivers 2 `.patch_0`/`.stream` Wwise archives with vgmstream (sourcing mod sound assets)
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

## Tooling Notes

- [ModForge UI Conventions](ModForge%20UI%20Conventions.md) — Compact modal sizing and typography conventions for the ModForge desktop app.
- [ModForge Managed Agent Docs](ModForge%20Managed%20Agent%20Docs.md) — How modman maintains the sentinel blocks in each mod's CLAUDE.md/AGENTS.md; refreshed only when a mod is opened, never via bulk sweep.
