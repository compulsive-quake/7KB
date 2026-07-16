# Map System (XUiC_MapArea)

How the in-game map (M) renders, why its zoom-out is hard-limited, and how to
show the whole world anyway. Working implementation: `Uncover/src/MapZoomPatches.cs`
(verified against the 2026-07 game build by decompiling `XUiC_MapArea` with ilspycmd).

## Rendering architecture

- The map widget is a 712x712 `XUiV_Texture` (`mapViewTexture` in
  `Data/Config/XUi_InGame/windows.xml`) with material `Materials/MaskableMainmap`.
- It displays `XUiC_MapArea.mapTexture`: a **2048x2048 RGBA32 scrolling ring
  buffer, 1 px = 1 block**, holding only a 2048x2048-block window around
  `mapMiddlePosChunks` (128x128 chunks; `updateMapForScroll` redraws 16px strips
  as you drag, `updateFullMap` repaints everything).
- The shader is driven by **global vectors set in `OnPreRender`**:
  `_MainMapPosAndScale` = (u, v, scale, scale) and `_MainMapBGPosAndScale` for
  the paper/fog background. Displayed UV = `mapPos + uv * mapScale`. All of it
  is computed in `positionMap()`:
  `mapScale = 336 * zoomScale / 2048` (336 = blocks across the view at zoom 1).
- Fog of war = alpha. A chunk absent from the player's map DB
  (`localPlayer.ChunkObserver.mapDatabase.GetMapColors(chunkKey)` → null) is
  written transparent; FOW soft edges come from `fow_chunkMask.png` alpha tiles.
  Chunk colors are `ushort` RGB555, decoded with `Utils.FromColor5To32`.
- Icons/cursor math never touches the texture: `worldPosToScreenPos` /
  `screenPosToWorldPos` only use `mapMiddlePosPixel` and `zoomScale`, so they
  keep working at any zoom.
- "Static map preview" mode (creative/debug button) builds
  `staticWorldTexture: Color32[]` covering the **entire world** at 1 px = 1 block
  (heights + water/roads from splat data, optional biome/radiation overlay) and
  routes it through the same 2048 window texture.

## Why zoom-out is limited (and the fix)

- Zoom is clamped to **[0.7, 6.15]** in two places with *inlined float consts*
  (`cMinZoomScale`/`cMaxZoomScale` are `const`, so patching the fields does
  nothing): mouse wheel in `onMapScrolled`, gamepad triggers in `Update`.
- 6.15 is not arbitrary: at that zoom the view spans 336 x 6.15 ≈ 2066 blocks —
  the whole 2048-block window. Raising the clamp alone shows wrapped garbage
  because the data outside the window simply isn't in the texture.
- Fix used by Uncover: raise the max (transpiler swapping `ldc.r4 6.15` for a
  computed `worldSpan * 1.05 / 336`; world size via `World.GetWorldExtent`) and,
  past zoom ~6, swap `xuiTexture.Texture` to a mod-built **downsampled
  whole-world overview texture** (1 px = 2/4/8/16 blocks so it stays ≤ ~2300px;
  built from the map DB, or by decimating `staticWorldTexture` in preview mode),
  positioned by writing the same `mapPos`/`mapScale` fields. Below the threshold
  rebind `mapTexture` and it's pure vanilla again.
- Gotchas that matter for that approach:
  - `XUiV_Texture.Texture` swap is safe only because the map widget doesn't set
    `autounload` (with `AutoUnload` the setter destroys the old texture).
  - Give the overview texture a transparent margin and clamp the view center to
    the world bounds — the shader may wrap UVs (the vanilla ring buffer needs
    it), so out-of-range UVs would tile the map at the edges.
  - Vanilla keeps `|mapMiddlePosPixel - mapMiddlePosChunks| < 16` via DragMap's
    while-loops. If you move `mapMiddlePosPixel` yourself (view clamping) or
    skip `updateMapForScroll` while the window is hidden (do — dragging at high
    zoom otherwise repaints the whole 2048 texture every frame), re-sync on the
    way back in with `PositionMapAt(pixelPos)` (recenters + full redraw).
  - `updateFullMap` fires every 0.5s while map data streams in
    (`IsNetworkDataAvail`) — a good hook (postfix) to know "map content changed".

## HUD minimap (bottom-right, IMGUI)

Uncover ships a live minimap (`Uncover/src/Minimap*.cs`, added 2026-07). Design
choices that worked out, so future minimap-ish work can reuse them:

- **Content comes from the same source as the big map**: the local player's
  `ChunkObserver.mapDatabase`. `MinimapTexture` keeps a **1024×1024-texel
  scrolling window texture** centered near the player, filled with the exact
  chunk-color decode the big map uses
  (`db.GetMapColors(WorldChunkCache.MakeChunkKey(cx,cz))` →
  `Utils.FromColor5To32`, `a=255`; unknown chunks cleared to `a=0` = fog). It
  recenters + full-rebuilds when the player strays past the recenter slack,
  refreshes the chunks around the player every ~0.75 s (only at 1:1), and
  full-refreshes (throttled) while an `uncover` reveal streams into the DB. Same
  data path as `MapZoomPatches.Overview` — just a smaller moving window.
- **Unlimited zoom-out via adaptive downsampling** (added 2026-07): the window
  is *not* fixed at 1 px = 1 block. A `blocksPerTexel` factor (power of two,
  1→16) is chosen from the current `ViewSpan` so the same 1024² texture always
  covers the view up to the whole world — the identical decimation trick
  `MapZoomPatches.Overview` uses (`pxPerChunk = 16/bpt`, sample one block per
  `bpt`). The old `MaxViewSpan = 600` cap existed *only* because a 1:1 1024
  window can't cover more than ~633 blocks (`RecenterSlack + viewSpan/2·√2 ≤
  512`); with downsampling the cap is now the **world extent** (`MinimapSettings.
  MaxViewSpanNow()` mirrors `MaxZoomScale`), capped by the bpt=16 ceiling
  (`AbsoluteMaxViewSpan ≈ 10k`). Key wiring: `bpt` and the recenter slack both
  scale with the factor; full renders `Array.Clear` the scratch then paint only
  the world-clamped chunk rect (a big window would otherwise scan empty margin);
  the **committed** bpt is stored with the origin so the widget's `Compose`
  samples with `sx = round((worldX − originX)/bpt)`, staying consistent with the
  pixels even while a rescale render is still in flight.
- **Rendering is IMGUI, not NGUI/XUi.** NGUI *can* rotate + circular-clip on
  the GPU (`UIPanel.clipping = SoftClip`/`TextureMask` + `clipTexture`;
  `UITexture.uvRect`/`OnFill` are public), but wiring a clipping UIPanel under
  XUi-managed transforms is fiddly and impossible to iterate on without seeing
  the screen. Instead a `MonoBehaviour.OnGUI` **composites** each frame's view
  into a small display `Texture2D` (cap the composite at ~224 px so cost is
  bounded regardless of on-screen size): for each display texel, rotate its
  centered offset by heading (rotate-on) or 0 (north-up), scale by
  `viewSpan/dim` blocks/texel, nearest-sample the source window, and multiply
  an analytic shape mask (round = `clamp01(radius − r)`, square = full with a
  1 px feather). Rebuild only when a signature changes
  (playerBlock, heading bucket, viewSpan, size, shape, source `ContentStamp`),
  throttled to 30 Hz — so with rotation off (default) it only recomposites on
  movement/zoom. `GetRawTextureData<Color32>()` on both textures (read source,
  write display) keeps it fast in Mono.
- **Orientation**: fill the composite with north at high texel-y (SetPixels32
  y=0 is bottom). `GUI.DrawTexture(rect, tex)` renders texel-top at rect-top,
  so north ends up at the top of the widget. Player arrow sprite
  (`ui_game_symbol_map_player_arrow` in the vanilla UI, or a procedural
  up-pointing triangle) at center: north-up mode rotates the arrow by camera
  yaw via `GUIUtility.RotateAroundPivot(yaw, center)` (positive yaw = clockwise
  = correct); heading-up mode leaves the arrow at 0 and rotates the map.
- **Heading source** = `localPlayer.playerCamera.transform.eulerAngles.y`
  (same value the compass binds as `{compass_rotation}`).
- **Settings persistence**: save runtime-tunable JSON under
  `GameIO.GetUserGameDataDir()` (…/7DaysToDie in %APPDATA%), **not** the mod's
  `Config/` — a `robocopy /MIR` deploy wipes files there that don't exist in
  source. `GameIO` also has `GetRoamingUserGameDataDir()` /
  `GetSaveGameDir()`; there is **no** `GetSaveGameRootDir`-free short alias
  besides `GetUserGameDataDir()`.

## Compass markers on the minimap (Uncover, 2026-07)

`Uncover/src/MinimapMarkers.cs` draws the compass's markers on the HUD minimap
by world position (off-screen ones clamped to the border), adds N/E/S/W to the
border, and shows outlined coords above it. Reusable facts:

- **The two marker sources the compass walks** (`XUiC_CompassWindow.Update`):
  1. `NavObjectManager.Instance.NavObjectList` (`List<NavObject>`) — traders,
     quest POIs, waypoints, tracked/party entities. Filter like the compass:
     skip `no.hiddenOnCompass || !no.IsValid()`, read
     `no.CurrentCompassSettings` (null → skip), gate on `MinDistance` /
     `no.GetMaxDistance(cs, player)` (`-1` = no cap). Sprite =
     `no.GetSpriteName(cs)`, color = `no.UseOverrideColor ? no.OverrideColor :
     cs.Color`. **NavObject positions are Origin-relative**: world XZ =
     `no.GetPosition() + Origin.position` (static `Origin.position`).
  2. `world.GetObjectOnMapList(EnumMapObjectType.X)` (`List<MapObject>`) for
     SleepingBag, LandClaim, MapMarker, MapQuickMarker, Backpack, Quest,
     TreasureChest, FetchItem, HiddenCache, RestorePower, SleeperVolume,
     VendingMachine, plus SupplyDrop when `GameStats.GetBool(
     EnumGameStats.AirDropMarker)`. Call `mo.RefreshData()`, gate on
     `mo.IsOnCompass()`, sprite = `mo.GetCompassIcon()` (fall back to
     `GetMapIcon()`), color = `mo.GetMapIconColor()`. **MapObject positions are
     absolute** world XZ (`mo.GetPosition()`) — no Origin offset, unlike
     NavObject.
- **Drawing an NGUI atlas sprite in IMGUI**: `xui.GetAtlasByName("UIAtlas",
  spriteName)` → `INGUIAtlas` (in `NGUI.dll`, so add that reference; returns
  null if the sprite isn't in that atlas). `atlas.texture` is the atlas
  Texture2D; `atlas.GetSprite(name)` → `UISpriteData {x,y,width,height}` in
  **top-left-origin pixels**. Convert to a bottom-left IMGUI UV rect (matches
  `NGUIMath.ConvertToTexCoords`): `new Rect(sd.x/tw, 1 - (sd.y+sd.height)/th,
  sd.width/tw, sd.height/th)`, then `GUI.DrawTextureWithTexCoords(rect, tex,
  uv)`. `GUI.color` tints (multiply) — set it to the marker color × fade alpha.
  Cache `(atlas, sprite)` by name; only cache resolved hits (atlases can still
  be loading early after spawn).
- **World offset → widget pixel** is the exact inverse of
  `MinimapTexture.Compose`. With `rad = rotate ? camYaw : 0` (radians),
  `cos/sin` of `rad`, `east = markerX - playerX`, `north = markerZ - playerZ`:
  `ox = east*cos - north*sin; oy = east*sin + north*cos`; screen offset from
  the widget center = `(ox * pxPerBlock, -oy * pxPerBlock)` where
  `pxPerBlock = rect.width / viewSpan` (+north is up = smaller screen y). Border
  clamp: round → `off * (limit / |off|)`; square → `off * (limit /
  max(|off.x|,|off.y|))`. Same transform places the cardinal letters (unit world
  dirs N=(0,1) E=(1,0) …) and, importantly, they rotate correctly with the map
  when heading-up is on.
- **"Move" vs "duplicate"**: to move the icons off the compass, Harmony-prefix
  `XUiC_CompassWindow.updateNavObjects` **and** `updateMarkers` returning
  `false` (both are publicized, patch by name). Skipping both leaves
  `waypointSpriteIndex` at 0, so the compass's own tail loop clears every marker
  sprite — the direction bar and clock stay. Gate the prefix on your minimap
  toggle so disabling the minimap restores the vanilla compass.
- **Outlined IMGUI text**: draw the label 8× in black at ±px offsets, then once
  in the fill color on top (`GUIStyle.normal.textColor` swapped between passes,
  `GUI.color.a` carries the fade).

## Minimap settings dialog (Uncover, 2026-07)

`Uncover/src/MinimapSettingsDialog.cs` is a self-contained **IMGUI** options panel
(same shape as `FPV/src/FPVOptionsDialog.cs`) that edits `MinimapSettings` directly,
so the minimap is fully configurable with no zPhone installed. It's opened by a gear
button injected into the map header (`XUiC_MinimapSettingsButton` +
`Config/XUi_InGame/windows.xml`). Reusable facts:

- **Close the map, don't overlay it.** The gear controller does
  `MinimapSettingsDialog.Open()` then `windowManager.Close("windowpaging")` (the same
  close the Uncover eye button uses). Running the IMGUI panel over the *closed* map,
  rather than on top of the open one, is deliberate for two reasons:
  1. **No click-through.** IMGUI has no colliders, so a panel drawn over the open map
     leaks drag/scroll/click to the map's NGUI widget underneath (pans/zooms it).
     `windowManager.IsInputLocked = true` does **not** fix this — per the
     `GUIWindowManager.Update` decomp it only gates the *global hotkey* loop and the
     Esc→pause path (`if (IsInputLocked) return;`), not NGUI UICamera collider events.
     Closing the map removes the collider entirely.
  2. **Live preview.** With the map closed the compass HUD comes back, so the
     bottom-right HUD minimap is visible and previews opacity/size/shape/zoom edits in
     real time while the dialog is up.
- **Modality for an IMGUI dialog over live gameplay** (map closed → world runs behind):
  `SetControllable(false)` **re-applied every frame** (closing the map restores control
  a frame or two later, so a one-shot freeze on open lets WASD move the "frozen"
  player — FPV's `FreezeForDialog` re-freezes each frame for the same reason);
  `IsInputLocked = true` each frame (blocks Esc→pause + toolbelt hotkeys);
  `Cursor.visible = true` / `lockState = None` each frame (the game re-locks it).
- **Do NOT `Input.ResetInputAxes()` each frame if you read `Input.GetKeyDown` for key
  rebinding or Esc.** ResetInputAxes zeroes button state for that frame and suppresses
  the very `GetKeyDown` reads a press-to-bind capture (and the Esc-to-close) depend on.
  `SetControllable(false)` already stops movement, so ResetInputAxes isn't needed.
- **Esc-close leaks to the pause menu** unless you keep `IsInputLocked` set for **one
  extra frame** after closing (the same Esc press is still `WasPressed` that frame and
  the window manager's Cancel→pause path would fire once you unlock). Defer the unlock
  to the next `Update` (tick it even while `!_open`).
- **No vanilla gear/cog sprite exists** (see Native Radial note). Ship a custom
  white-on-transparent PNG in `UIAtlases/ItemIconAtlas/` (e.g.
  `ui_uncover_minimap_settings.png`, 128×128) and reference it with
  `atlas="ItemIconAtlas"`. New atlas PNGs are **baked at load → need a restart**, not
  `xui reload`. Generated the cog procedurally with PowerShell + `System.Drawing`
  (polygon whose radius alternates rTip/rRoot per tooth, plus an even-odd center-hole
  ellipse for the donut).
- **Map header x-layout** (for future header injections; `rect[@name='header']` in
  `Data/Config/XUi_InGame/windows.xml`): `switchStaticMap` is at **690,-21**,
  `cbxStaticMapType` at **690**-ish default (210 wide, only visible in static-map mode).
  Uncover packs three icons left of switchStaticMap: gear **600**, eye **652**, and
  moves the combobox to **360** (spans 360..570) so it clears both.

## Related facts

- The full-map reveal itself (fog removal) is a separate concern: see
  `Uncover/plan.md` — vanilla `visitmap` generates chunks but never writes the
  per-player fog DB; you must `mapDatabase.Add(x, z, chunk.GetMapColors())`.
- Assembly-CSharp ships **publicized** (`[PublicizedFrom]` attributes): private
  members of `XUiC_MapArea` (`zoomScale`, `mapPos`, `positionMap`, ...) are
  public in the DLL, so patches can use them directly — no reflection, and
  `nameof()` works at compile time.
- Harmony lives at `Mods/0_TFP_Harmony/0Harmony.dll` (NOT in
  `7DaysToDie_Data/Managed`) — reference it with `Private=false`.
