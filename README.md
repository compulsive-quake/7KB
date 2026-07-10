# 7 Days to Die Modding Knowledgebase

Shared knowledgebase for 7DTD mod development. Used by multiple projects (7nes, 7builder, PixelPaste).

## Modding Fundamentals

- [Mod Structure](Mod%20Structure.md) — Folder layout, ModInfo.xml, DLL loading, modlet system
- [XML Patching (XPath)](XML%20Patching%20(XPath).md) — How to patch vanilla XML configs; sounds system
- [Harmony Patching](Harmony%20Patching.md) — Harmony prefix/postfix patterns, common targets, reflection
- [7DTD 3.0.0 API Changes](7DTD%203.0.0%20API%20Changes.md) — Breaking 2.x→3.0.0 changes: **XUi folder split (`Config/XUi/` → `Config/XUi_InGame/` or mod UI silently doesn't load → `Window unknown!`)**, `<ruleset>` wrapper removed from `xui.xml`, `World` `clrIdx` param removal (`GetTileEntity`/`SetBlockRPC` now take `BlockValueRef`), `ChunkClusters`→`ChunkCache`, `RefreshBindings()` arg drop, `GUIWindowManager.Open` overloads; recompile to surface the C# breaks as compile errors
- [Localization](Localization.md) — Adding translated text keys
- [Blocks](Blocks.md) — Custom block classes, XPath patches, block properties
- [Structural Integrity & Falling Blocks](Structural%20Integrity%20%26%20Falling%20Blocks.md) — Stability channel internals (0–15), the `World.AddFallingBlock` choke point, and the two-layer recipe (stability-15 stamp + Harmony prefix) for keeping runtime-moved blocks from collapsing
- [Moving Blocks at Runtime (SetBlocksRPC)](Moving%20Blocks%20at%20Runtime%20(SetBlocksRPC).md) — Relocating world blocks in batches (capture → TE detach → `SetBlocksRPC` → TE reattach) and rendering a `BlockValue` as a smooth-gliding proxy GameObject via `ItemClassBlock.CreateMesh` (elevators, moving platforms); rotation/paint baked from the real BlockValue, `ToItemValue()` strips rotation, layer 16, `Origin.position`
- [Moving Platforms & Riding Entities](Moving%20Platforms%20%26%20Riding%20Entities.md) — Making a runtime-moved collider carry players/zombies/vehicles without blocking their movement: full layer table + collision-matrix facts (13 “Items” touches NO character controller; 16 “TerrainCollision” is ground to everyone), why every fixed-step approach (MovePosition/FixedUpdate, the dormant UFPS `m_Platform` code) stair-steps the camera and looks shaky, the butter-smooth per-render-frame recipe (state-preserving `vp_FPController` shift for the local player, suspension physics for driven vehicles), and the filtered vertical-delta carry for KCC entities (`InteractiveRigidbodyHandling=false`) and parked (voxel-AABB) vehicles
- [Workstations](Workstations.md) — Workstation blocks, modules, tool slots, XUi window groups
- [Items](Items.md) — Custom item definitions, icons (ItemIconAtlas), dynamic item generation
- [Power Sources](Power%20Sources.md) — Custom power generators, TileEntityPowerSource, fuel management, power system C# internals
- [Entities](Entities.md) — Custom entity classes (NPCs, zombies, animals), AI tasks, drops
- [Vehicles](Vehicles.md) — Custom vehicles (drivable mounts, hover bikes), `vehicles.xml` schema, mount/dismount pipeline, seat poses, camera switch, scripted-locomotion gotchas
- [Networking - Connecting and Chat](Networking%20-%20Connecting%20and%20Chat.md) — ConnectionManager, sending chat via NetPackageChat, JSON escape gotcha in 7debug's /api/command
- [Asset Bundles](Asset%20Bundles.md) — Loading custom models, particles, sounds from .unity3d bundles
- [Runtime Procedural VFX](Runtime%20Procedural%20VFX.md) — Runtime LineRenderer/particle effects with no bundle; the `HideAndDontSave` texture/material gotcha, procedural lightning, world→scene space
- [Runtime Explosions (ExplosionData)](Runtime%20Explosions%20(ExplosionData).md) — `GameManager.ExplosionServer` code-driven blasts; the 3.0.0 silent-no-op gotcha (`ExplosionData` reads nested `Classes["Explosion"]` keys, not flat `Explosion.*`) and the `ExplosionServer` `clrIdx` param removal
- [Player Feedback Sounds](Player%20Feedback%20Sounds.md) — Stock UI "denied" buzzers (e.g. `missingitemtorepair`) and how to play them via `Audio.Manager.PlayInsidePlayerHead`
- [Runtime WAV Loading (AudioClip from disk)](Runtime%20WAV%20Loading%20(AudioClip%20from%20disk).md) — Ship a loose `.wav` in `Resources/` and decode it into an `AudioClip` at runtime (RIFF chunk-walk + PCM→float + `AudioClip.Create`/`SetData`) with no asset-bundle rebuild; see `FPV/src/FPVWavLoader.cs`
- [Synthesized Drone Rotor Audio](Synthesized%20Drone%20Rotor%20Audio.md) — Procedural racing-quad sound: RPM→pitch model (Unity `AudioSource.pitch` clamps at ±3), fundamental-dominant harmonics to avoid rasp, two-pole-filtered prop-wash noise, and an ffmpeg+numpy workflow for tuning a synth against a real recording; see `FPV/src/FPVDroneAudio.cs`
- [Remote Camera World Loading (Drone FPV)](Remote%20Camera%20World%20Loading%20(Drone%20FPV).md) — Loading + rendering the world around a camera far from the player body (FPV drone / spectator). Three independent player-centric systems: a **ChunkObserver** loads voxels/collision (easy — matches vanilla's own player observer), but **entity render-fade** (`World.SetEntitiesVisibleNearToLocalPlayer` → `EntityAlive.VisiblityCheck` fades models past 90 m from the *player* camera → real-but-invisible zombies) and **Distant-POI DynamicMesh imposters** (marble no-collision husks; the hide rule `!IsChunkInGame` only fires near the player) each need their own drone-aware nudge
- [Secondary Camera Feeds (RenderTexture + IMGUI)](Secondary%20Camera%20Feeds%20(RenderTexture%20%2B%20IMGUI).md) — Extra in-world cameras (drone feed, pilot PiP): RenderTexture + `GUI.DrawTexture` pattern, **clone the main camera's `PostProcessLayer` or the feed renders dark/flat** (private `m_Resources` via reflection), PiP of the still-rendering main view by mirroring `Camera.main` pose in LateUpdate, per-camera layer hiding via `cullingMask`
- [Camera View Switching (First/Third Person)](Camera%20View%20Switching%20(First-Third%20Person).md) — `SetFirstPersonView`/`SetCameraAttachedToPlayer` internals; `playerCamera.transform` IS `cameraTransform`; the cross-mod bug where a temporary view switch leaves the camera + held-item transforms parked and breaks other mods' laser/aim (cone beam, targets own head), and the multi-frame re-assert fix
- [Escape Menu Pause & World-Save Stall](Escape%20Menu%20Pause%20%26%20World-Save%20Stall.md) — ESC menu `OnOpen` → `Pause(true)` → single-player `SaveWorld` → `RegionFileManager.WaitSaveDone()` spins the main thread flushing dirty chunks (1–2s freeze after a roaming ChunkObserver); the log tell (`Saving N of chunks took Xms`), why close-after-open can't fix it, and the Harmony-free `HudEnabledStates.FullHide` gate (defer HUD restore until ESC releases)
- [Paint & Textures](Paint%20%26%20Textures.md) — Block paint texture IDs and the paint system
- [Time & Weather Control](Time%20%26%20Weather%20Control.md) — Freeze/set the clock (`GameStats TimeOfDayIncPerSec`, 0 = frozen, default `24000/(DayNightLength*60)` re-derived every world start), `WeatherManager.force*` fields + off sentinels (`-1f` / temp `-100f`, `BaseTemperature`=70°F), why both need a re-apply enforcer after world load, and blocking player status effects via an `EntityBuffs.AddBuff` prefix
- [Reading HID Joysticks (hid.dll)](Reading%20HID%20Joysticks%20(winmm).md) — read RC transmitters/joysticks via setupapi + hid.dll (legacy Input Manager can't); full 8-axis recipe, plus why winmm is a dead end (only 6 slots — HID Ry and Dial are invisible to it)
- [Betaflight Rates Math](Betaflight%20Rates%20Math.md) — Faithful Betaflight rateprofile formulas (RC Rate/super rate/expo, throttle MID/EXPO + limit, TPA) for sim flight models; the pre-expo `|stick|` superfactor gotcha, max-vel formula, cached-texture IMGUI curve previews, and the FPV csproj `GameDir` build flag
- [Chunk Observers & Chunk Loading](Chunk%20Observers%20%26%20Chunk%20Loading.md) — `AddChunkObserver` API for loading/displaying terrain around a non-player viewpoint (camera drone); viewDim caps, SP vs MP-client gating, why the whole pipeline is observer-driven (no camera/player gating), the fast-mover trailing-bubble problem (lead the observer), and the vanilla `chunkobserver` console command + `Time:` log line decoding
- [Unity Sweep Casts & Resting Contact](Unity%20Sweep%20Casts%20%26%20Resting%20Contact.md) — **PhysX box/sphere sweeps report already-overlapping colliders as `distance = 0` hits with `point = (0,0,0)`, `normal = -direction`**; custom flight physics that bounces/rests on any hit freezes an object resting in ground contact (can't take off even straight up). Fix: skip `distance <= 0` hits in movement sweeps + park grounded craft on a small virtual floor
- [Procedural Texture Fills & Main-Thread Stalls](Texture2D%20Pixel%20Writes%20(GetRawTextureData%20vs%20SetPixels32).md) — a 1M-texel minimap fill measured ~4.5–5s on the main thread at spawn **regardless of write path** (NativeArray vs managed+`SetPixels32` — the write path was NOT the cost); never run big texture composition on the main thread — worker-thread render + coalesced jobs + `SetPixels32` upload pattern. Includes the frame-gap + GC-delta + Harmony-stopwatch stall-probe recipe for attributing silent main-thread freezes
- [Voxel Light & Scene Brightness](Voxel%20Light%20%26%20Scene%20Brightness.md) — **`World.GetLightBrightness` is static sky *exposure*, not current brightness** (1.0 outdoors at midnight, ~0 under a bridge at noon); correct recipe = `max(Chunk.GetLight(BLOCK)/15, SkyManager.GetSunIntensity())`; SkyManager statics reference
- [Map Fog & Reveal](Map%20Fog%20%26%20Reveal.md) — Map fog-of-war = which chunks are in the per-player `ChunkObserver.mapDatabase` (no hide flag), written only by `EntityPlayer.Update` in a 9×9 area around the player. **`visitmap full` does NOT reveal fog** (it pregens chunks + caches colors for the world-tile renderer). Correct reveal = `MapVisitor` to load every chunk + explicit `mapDatabase.Add(c.X, c.Z, c.GetMapColors())`. Mod console commands auto-register via reflection (**no Harmony**), but `ConsoleCmdAbstract` overrides must be `public` (CS0507 gotcha). **A fully-populated fog DB freezes spawn ~9s**: `MapChunkDatabaseByRegion.Load` holds `m_regionsLock` the whole load and the main thread blocks on it first frame after spawn — fix by bulk-decompress + per-region locking (Uncover `MapDbLoadPatch`)

## XUi (UI System)

- [XUi - Window System](XUi%20-%20Window%20System.md) — XML window definitions, elements, colors, styles, controls
- [XUi - Controllers (C#)](XUi%20-%20Controllers%20(C%23).md) — C# XUiController subclasses, bindings, events
- [HUD Safe Zones](HUD%20Safe%20Zones.md) — Top DAY/TIME strip + bottom message/toolbelt strip must not be overlapped by IMGUI or non-HUD-hiding windows
- [IMGUI Tracker Stack](IMGUI%20Tracker%20Stack.md) — Right-side stacked status cards for tracked entities (HP bar + status text + fade in/out). Also covers interactive IMGUI tool overlays (alignment / tuning sliders): always include a typeable text field next to step buttons, expose `static IsOpen` so other input listeners can short-circuit (`Event.current.Use()` doesn't block `Input.GetMouseButtonDown`).
- [zPhone External Apps](zPhone%20External%20Apps.md) — External app manifests, JSON page controls, action bridge registration, and held-item tuning patterns.
- [Embedded Edge Capture (zPhone WebSurface)](Embedded%20Edge%20Capture%20(zPhone%20WebSurface).md) — Hidden `msedge --app` + WGC capture + CDP input pipeline; **Edge chrome height is not constant** (injected infobars like the "browser data" banner shift the page and break fixed-offset click mapping — measure `window.innerHeight` via the bridge instead), plus user-data-dir/job-object/throttling gotchas.
- [Native Radial Menu (XUiC_Radial)](Native%20Radial%20Menu%20(XUiC_Radial).md) — Drive the vanilla radial wheel from a mod via `xui.RadialWindow` (no XML); hold-key-to-select comes free; custom gear/icon sprites; the 2026-06-29 `ExplosionServer`/`clrIdx` API change.

## Prefab & Binary Formats

- [TTS File Format](TTS%20File%20Format.md) — TerraTerrain Storage binary format (voxel grid)
- [NIM File Format](NIM%20File%20Format.md) — Name Index Map binary format (block ID → name)
- [Pregen World Format](Pregen%20World%20Format.md) — `Data/Worlds/` folder layout; `main.ttw` header gotcha
- [Extracting FMOD Bank Audio (FSB5) & Custom Vehicle Sounds](Extracting%20FMOD%20Bank%20Audio%20(FSB5)%20%26%20Custom%20Vehicle%20Sounds.md) — Carving PCM16 WAVs out of FMOD Studio .bank files (Assetto Corsa soundsets) and wiring them as a vehicle's engine/horn sounds via sounds.xml + bundle clips
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
- [Renaming a Mod (modman)](Renaming%20a%20Mod%20(modman).md) — Full rename checklist (contents, git mv, gh repo rename + SSH-remote gotcha, stale DLLs, old `Mods/` copy); why the folder itself can't be renamed while a modman Claude session is open (session cwd handles) and what to do instead.
