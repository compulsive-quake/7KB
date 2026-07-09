# Remote Camera World Loading (Drone / FPV / Spectator)

Making the world load **and render** around a camera that is far from the local
player's body (an FPV drone feed, a spy camera, a spectator cam). 7DTD keys many
subsystems off `EntityPlayerLocal`, so pointing a camera somewhere else is not
enough — three **independent** systems must each be told about the remote point.
Symptoms map cleanly to which one you've missed.

## 1. Voxel chunks — solved by a ChunkObserver (this one is easy)

`world.m_ChunkManager.AddChunkObserver(worldPos, _bBuildVisualMeshAround: true, viewDim, -1)`
loads + meshes real voxel chunks (terrain, POI blocks, collision) around any
point. This is the **exact** call vanilla makes for the local player
(`GameManager` line ~2104/3395), so an observer at the drone gives full terrain
with collision out there. Update it with `observer.SetPosition(worldPos)` each
frame; `RemoveChunkObserver` on teardown.

- Coordinates are **absolute world** (matches `Entity.position`,
  `player.GetPosition()`); the Unity transform is `worldPos - Origin.position`.
- `viewDim` clamp: `rectanglesAroundPlayers` has 15 rings and an observer uses
  `viewDim + 2`, so **viewDim ≤ 13**; vanilla clamps client observers to 12.
- `entityIdToSendChunksTo = -1` = the host/integrated-server observer (builds
  visual mesh locally). A real entityId is for sending chunks to a networked
  client.
- Ground-truth telemetry: iterate `world.ChunkCache.GetChunkSync(MakeChunkKey(cx,cz))`
  around the drone and count `chunk.IsDisplayed`. `49/49 displayed` at 280 m from
  the body proves the observer works — so if things still look wrong, it's #2/#3,
  **not** the observer. (In `FPV`: `EnsureDroneChunkObserver` / `LogChunkDiagnostics`.)

## 2. Entities invisible except HUD box — render-fade cull (player-centric)

Symptom: a zombie is *real* (AI ticks, chases the drone) but invisible in the
feed — only a HUD marker. Cause: `World.SetEntitiesVisibleNearToLocalPlayer()`
measures every entity's distance from **`primaryPlayer.cameraTransform.position + Origin.position`**
and calls `EntityAlive.VisiblityCheck(distSqr, isZoom)`, which sets
`renderFadeTarget = 0` (fade model to invisible) beyond **8100 = 90 m²**
(14400 = 120 m² when zooming). Runs every tick, so it re-hides continuously. Only
acts on entities with `EntityFlags.AIHearing` (zombies/animals).

Fix: Harmony **Postfix** on `SetEntitiesVisibleNearToLocalPlayer`; while the feed
is live, re-run `entity.VisiblityCheck((entity.position - droneEye).sqrMagnitude, false)`
for entities within 90 m of the **drone camera**. Call it only with an in-range
distance so it can only turn a model **on** — anything out of the drone's range is
left exactly as vanilla decided, so you never hide what the player pass showed.
Skip `EntityPlayerLocal`. `World.Entities` is a `DictionaryList<int,Entity>`;
iterate `.list`. (In `FPV`: `src/FPVFeedVisibilityPatch.cs`.)

## 3. Buildings are marble husks you fly through — POI imposters (player-centric)

Symptom: distant buildings wear a coarse **marble** shell with **no collision**
(drone passes through); real chunk geometry and the husk can even render
simultaneously (real windows poking through marble slabs); the shell doesn't
exist when you walk there in person.

**There are TWO separate husk systems — identify before patching** (a
renderer-bounds probe at the crosshair settles it, see §4):

### 3a. PrefabLODManager — the POI building imposters (THE marble buildings)

The baked POI imposters live under a **`PrefabsLOD/<poiName>/<x,z>`** GameObject
hierarchy, shader **`Game/DistantPOI_TA`**, one child per **32 m tile** named in
prefab-local tile coords (`"-1,0"` etc.). Managed by `PrefabLODManager`
(`FrameUpdate` ~1 Hz → `UpdateDisplay(localPlayer)`):

1. Scans displayed chunks within **192 m of the LOCAL PLAYER** (`36864 = 192²`,
   `IsDisplayed && IsCollisionMeshGenerated`) and reduces them to ONE axis-aligned
   rectangle (trimming incomplete edge rows).
2. POI bounding box entirely outside that rect → **all tiles shown** (guarded by
   `PrefabGameObject.isAllShown`). Overlapping → tiles **fully inside** the rect
   are `SetActive(false)` (real chunks render there), others shown.

A drone's chunk observer displays real chunks far outside that scan radius, so
the rect never covers the drone's area → imposter tiles render **on top of** the
real loaded geometry (both draw at once).

**`UpdateDisplay` CANNOT be Harmony-patched**: MonoMod fails to re-JIT its IL —
`HarmonyException: IL Compile Error … Label #45 is not marked in method
DMD<PrefabLODManager:UpdateDisplay>` — and the exception thrown from `PatchAll`
aborts the mod's whole `InitMod` (see [[Harmony Patching]]). Fix instead: rerun
the identical logic from your own **`LateUpdate`** (same rect scan + edge trim,
same tile-name parse + `PrefabInstance.rotation` mapping) with the rect computed
around the **drone**, hiding tiles fully inside it, and set `isAllShown = false`
on any prefab touched so vanilla's early-out re-shows tiles after the feed ends.
Sync to vanilla: it runs from Update at ~1 Hz and stamps `mgr.lastDisplayUpdate`;
run your pass when that stamp changes (LateUpdate is after Update in the same
frame → its re-shows are corrected before rendering, no flash) plus when the
drone crosses a chunk. Instance: `GameManager.Instance.prefabLODManager`. (In
`FPV`: `src/FPVPrefabLODPatch.cs` helper + `RunPrefabLodDronePass` in
`FPVDroneManager.cs`.) Note `PrefabLODManager` also maintains
`prefabsAroundNear/Far` per player (imposters exist only for POIs outside the
player's chunk view, within `lodPoiDistance` 1000 m).

### 3b. DynamicMesh — player-modified-chunk imposters

The **DynamicMesh** system (`DynamicMeshManager`) husks are a different layer.
Its architecture:

- One combined imposter mesh per **160 m region** (`DynamicMeshRegion.RegionObject`),
  split into per-chunk **item** meshes (`DynamicMeshItem.ChunkObject`) near a player.
- **Hiding over real geometry is EVENT-driven and position-agnostic**: every chunk
  display (`Chunk.OnDisplayBlockEntities`) calls
  `DynamicMeshThread.AddChunkGameObject` → adds to `ChunkGameObjects` (backs
  `item.IsChunkInGame`), `item.ForceHide()`, and `region.OnChunkVisible` →
  `HideRegion()` (hides the combined husk). This fires for drone-observer chunks
  too — so far so good.
- **BUT the recurring pass re-shows it**: `DistanceChecks()`/`SetViewStats()` runs
  constantly over all regions, and `if (!inBuffer && !isOutsideMaxRegionArea)
  SetVisibleNew(true)` re-shows the region husk for any region **not within ±1
  region of the PLAYER** (`IsInBuffer()`), even over fully displayed drone chunks.
  The event path hides; the periodic pass re-shows a tick later — marble wins.
  Naively hiding item meshes doesn't help: the visible husk is the REGION mesh.

Every predicate in that pipeline reads `GetPrimaryPlayer().position`:
`IsInBuffer()` (±1 region), `IsInItemLoad()` (±3), `IsInItemUnload()` (outside ±4
→ `ClearItems`), `IsPlayerInRegion()` (gates `HideRegion` + blocks `SetVisibleNew(true)`),
`DistanceToPlayer()` (region + item, unload/CleanUp/load-priority), and
`DynamicMeshItem.IsChunkInView` (player chunk-view square; freshly loaded items
show only outside it).

Fix — **make the drone count as a second player presence**: tiny Harmony postfixes
on each predicate that OR-in (or min-in) the drone position while the feed is
live; each conveniently has a public position-parameterized overload to call
(`IsInItemLoad(float,float)`, `DynamicMeshUnity.IsInBuffer(x,z,buffer,xIdx,zIdx)`,
`item.DistanceToPlayer(x,z)`, `DynamicMeshUnity.RoundRegion`). Then the vanilla
pipeline (load items, hide region, hide items over real chunks, re-show on chunk
unload) runs natively around the drone. **Threading**: some of these run on the
dymesh worker thread — patches must read a plain-float cached position, never
Unity APIs. Plus one level-triggered reconciliation (edge-triggered
`OnChunkVisible` misses regions whose chunks displayed *before* the drone arrived):
periodically, for each `DynamicMeshRegion.Regions.Values` in a viewer's buffer
with ≥1 `item.IsChunkInGame`, call `region.HideRegion(reason)` (which also runs
`ShowItems()` = per-item `SetVisible(!IsChunkInGame)`). All members are public on
`Assembly-CSharp` — no reflection. (In `FPV`: `src/FPVDynamicMeshPatches.cs` +
`HideImpostersOverLoadedChunks` in `FPVDroneManager.cs`.)

`DynamicMeshManager` also has `CONTENT_ENABLED` / `EnabledChanged(bool)` as a
global kill switch and a `DisabledImposterChunkManager` shader-clip buffer
(`PREFAB_CLIPPING_ON`, clips region-mesh fragments per-chunk where item meshes
exist) — heavier levers than the presence patches.

## 4. Debugging technique — the renderer-bounds crosshair probe

When a mystery mesh renders where it shouldn't, don't guess its owner from
static analysis — the husk systems look identical on screen and there are
several (PrefabLODManager, DynamicMesh region + item meshes, UnityDistantTerrain).
Instrument instead: every few seconds, take probe points along the camera
forward ray (e.g. 4/10/24/48 m) and log every active `Renderer` whose
`bounds.SqrDistance(point) < 1f` — full transform path, layer, shader,
material, bounds size (`FindObjectsOfType<Renderer>()` is fine at 0.2 Hz).
The transform path + shader name identify the owner beyond doubt (e.g.
`PrefabsLOD/house_modern_18/-1,0 | Game/DistantPOI_TA` vs
`Chunk_132,-9/CLayer02/opaque | Game/Overlayed_TA`). This cracked the FPV
marble-husk case after two plausible-but-wrong dymesh fixes. Rebuild it in a few
lines when needed: probe points along `camera.forward`, `FindObjectsOfType<Renderer>()`,
filter `enabled && activeInHierarchy && bounds.SqrDistance(p) < 1f`, log
transform path + `sharedMaterial.shader.name` (temporary code — remove after).

## Key takeaway

Chunk data (#1) and the **render** gates (#2 entities, #3a POI imposters,
#3b dymesh) are separate systems. A working ChunkObserver makes the world
*exist* around the drone but does **not** make far entities or imposter husks
render correctly — those are culled/swapped by distance to the **local player**,
and each needs its own drone-aware nudge. When the on-screen evidence and your
model disagree, reach for the renderer probe (#4) before writing another fix.
Related: [[Entities]], [[Camera View Switching (First-Third Person)]],
[[Map Fog & Reveal]] (also `ChunkObserver`-driven), [[Escape Menu Pause & World-Save Stall]],
[[Secondary Camera Feeds (RenderTexture + IMGUI)]] (rendering the feed itself + pilot PiP).
