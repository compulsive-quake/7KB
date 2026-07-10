# Map Fog & Reveal (fog-of-war)

Part of the [7DTD Modding Knowledgebase](README.md). Covers how the in-game map
(opened with **M**) stores fog-of-war and how to reveal it. Verified against the
Steam client build present in July 2026 (Uncover mod).

---

## How the fog works

The map fog is **not** a hide flag. The map is drawn per-player from a chunk
color database; a chunk that has a stored 16×16 color strip is drawn, a chunk
that has none is fog. There is no way to "re-hide" a known chunk short of
clearing the database.

- Per-player DB: `player.ChunkObserver.mapDatabase` (`IMapChunkDatabase`).
  Implementations: `MapChunkDatabase` (Fixed) / `MapChunkDatabaseByRegion`
  (Region) — chosen by `LaunchPrefs.MapChunkDatabase`. Region variant stores
  `ushort[256]` per chunk (16×16), 32×32 chunks per region file `r.X.Y.7rm`
  (gzip). `GameManager.fowDatabaseForLocalPlayer` also points at the local
  player's DB.
- Renderer: `XUiC_MapArea` reads `mapDatabase.GetMapColors(chunkKey)` for each
  chunk and uses `mapDatabase.Contains(neighborKey)` to draw fog edges. So fog =
  absence of that chunk in the DB.
- Colors come from `Chunk.GetMapColors()` (top-block colors of a decorated
  chunk). A chunk must be **generated + decorated** to have colors, so you can't
  reveal ungenerated terrain without generating it.
- Populate path: on load, the server-side observer queues map pieces
  (`ChunkManager` → `GetMapChunkPackagesToSend`) and sends `NetPackageMapChunks`,
  whose `ProcessPackage` calls `mapDatabase.Add(chunks, mapPieces)` on the
  client. In single-player the integrated host does the same in-process.

---

## Who actually clears fog (and who doesn't)

The **only** vanilla writer of the per-player fog DB is `EntityPlayer.Update`:
when the player's `chunkPosAddedEntityTo` changes it calls
`ChunkObserver.mapDatabase.Add(chunkPos, world)`, and that default `Add(Vector3i,
World)` overload reveals a **9×9 chunk area** (`num = editor ? 7 : 4`, loop
`-num..num`) around the player's *own* position. That's why the map fills in as
you walk.

> **`visitmap full` does NOT reveal the player's map fog.** `ConsoleCmdVisitMap`
> runs a `MapVisitor` and its callback only calls `chunk.GetMapColors()`, which
> *caches colors on the Chunk object* for the separate world-tile renderer
> (`MapRendering.MapRenderer`) and runs optional density checks. It never calls
> `mapDatabase.Add`, so the M-map fog is untouched. It's a dev pregen/tile tool,
> not a fog-reveal. There is no built-in "reveal entire map" command — the only
> `mapDatabase.Add` callers are `EntityPlayer.Update` (9×9 around the player) and
> `NetPackageMapChunks` (client receiving from server).

## Revealing the whole map (correct recipe)

Combine `MapVisitor` (to load/generate + decorate every chunk, so real colors
exist) with an explicit `mapDatabase.Add` per chunk — the step visitmap omits:

```csharp
EntityPlayerLocal player = GameManager.Instance.World.GetPrimaryPlayer();
IMapChunkDatabase db = player.ChunkObserver.mapDatabase;   // == fowDatabaseForLocalPlayer
GameManager.Instance.World.GetWorldExtent(out Vector3i min, out Vector3i max);

var visitor = new MapVisitor(min, max);   // force-loads a ChunkObserver over the rect
visitor.OnVisitChunk += (c, done, total, t) =>
    db.Add(c.X, c.Z, c.GetMapColors());   // <-- the crucial fog write
visitor.Start();                          // coroutine; visitor.Stop() to abort
```

`MapVisitor.visitCo` only fires `OnVisitChunk` once a chunk is loaded AND
`!NeedsDecoration`, so `GetMapColors()` returns real terrain colors.
`IMapChunkDatabase.Add(int cx, int cz, ushort[])` just stores the 256-color strip
into region data (does not need the chunk to stay loaded), which is what
`XUiC_MapArea` reads. Genuinely **slow and CPU/disk heavy** on large maps (every
chunk gets generated) — log progress and warn the user; re-open the map (M) to
see the result. This is Uncover's `uncover` command.

---

## Spawn freeze after a full reveal (fog DB load holds its lock)

Once the fog DB is fully populated, clicking Spawn freezes the game for many
seconds (~9s on a 6k map / 169 region files). Log tell: right after
`PlayerSpawnedInWorld` there is a silent gap ending in
`ChunkManager mesh regeneration thread resumed after N.Ns blocked` (the VML
queue backs up because the main thread is stalled).

Cause (verified against the July 2026 build, `MapChunkDatabaseByRegion`):

- `IMapChunkDatabase.TryCreateOrLoad` schedules `LoadAsync` via
  `ThreadManager.AddSingleTask` when the player's chunk observer is created at
  spawn — background thread, so far so good.
- But `Load(rootDirectory)` holds `m_regionsLock` for the **entire** load, and
  parses each `r.X.Y.7rm` with 262,144 individual `ReadUInt16` calls through a
  `GZipStream` plus a LINQ `SequenceEqual(EMPTY_CHUNK_DATA)` per chunk —
  ~50ms+ per region on Mono.
- The main thread blocks on that same lock on its first frame after spawn:
  vanilla `EntityPlayer.Update` → `mapDatabase.Add(chunkPos, world)` (the 9×9
  reveal) takes it, as does any `GetMapColors`/`Contains` (map, minimap).

On a normal save only a few region files exist so nobody notices; a full
reveal makes every region exist and turns the load into a multi-second lock
hold = main-thread freeze.

Fix (see `MapDbLoadPatch` in the Uncover mod): Harmony-prefix-replace `Load` to
(1) bulk-decompress each region into a reusable 512KB buffer and convert chunks
with `Buffer.BlockCopy` instead of per-ushort reads, and (2) parse regions
OUTSIDE the lock, locking only for the per-region dictionary insert. Load drops
to well under a second and the main thread never waits. File format: 4-byte raw
version header, then gzip of 32×32 chunks × 256 little-endian ushorts; an
all-zero 512-byte chunk slice means unexplored (slot stays null = fog). Set
`m_dirty = false` / `m_lastRegionPath` on the rebuilt `RegionData` or the next
save rewrites every region. `Save` also holds the lock across all regions but
skips clean ones cheaply, so it's only slow right after the reveal itself.

The original ~9s freeze turned out to be TWO stacked causes: the lock hold
above, plus the HUD minimap's first full-window redraw writing ~1M texels
through `GetRawTextureData<Color32>()` (~5s by itself) — see
[Texture2D Pixel Writes (GetRawTextureData vs SetPixels32)](Texture2D%20Pixel%20Writes%20(GetRawTextureData%20vs%20SetPixels32).md).

---

## Console command from a mod DLL (no Harmony needed)

`SdtdConsole.RegisterCommands` calls
`ReflectionHelpers.FindTypesImplementingBase(typeof(IConsoleCommand), …)` across
**all loaded assemblies**, so a mod just needs a public, parameterless-ctor class
extending `ConsoleCmdAbstract`; it auto-registers. No Harmony, no `IModApi`
wiring required for the command itself.

- Command is invoked in the F1 console **without** a leading slash: `uncover`,
  not `/uncover` (the slash form is chat commands).
- **`ConsoleCmdAbstract`'s overridable methods are `public`, not `protected`, in
  the shipped (publicized) `Assembly-CSharp`.** The DLL tags them
  `[PublicizedFrom(EAccessModifier.Protected)]` — decompilers show the original
  `protected`, but the actual metadata is `public`. Declaring
  `protected override string[] getCommands()` fails with **CS0507 "cannot change
  access modifiers"**. Use `public override` for `getCommands` / `getDescription`
  / `getHelp`.

---

## Adding a button to the in-game map (M) header

The map is the `mapArea` window in `Config/XUi_InGame/windows.xml` (controller
`XUiC_MapArea`). Its header rect (`rect[@name='header']`) holds the vanilla
`switchStaticMap` icon button (`ui_game_symbol_electric_switch`, pos `690,-21`)
and the `cbxStaticMapType` combobox (pos `460,-6`, only visible in static-map
mode). To add your own button next to it, `insertAfter` the `switchStaticMap`
button and nudge the combobox left (`set .../combobox[@name='cbxStaticMapType']/@pos`)
so nothing overlaps in static mode.

**Creative/debug gate (verified against the July 2026 Steam build).** The vanilla
`switchStaticMap` button is shown only for a host in creative or debug mode:

```csharp
switchStaticMap.IsVisible =
    SingletonMonoBehaviour<ConnectionManager>.Instance.IsServer
    && (GameManager.Instance.IsEditMode()
        || GamePrefs.GetBool(EnumGamePrefs.DebugMenuEnabled));
```

Reuse the exact same condition to match "creative and/or debug mode". (On a
dedicated-server client `IsServer` is false; a client can't force-generate chunks
to reveal fog anyway, so hiding it there is also correct.)

**Closing the map from a controller.** The map is hosted inside the
`windowpaging` pager (the `map` window_group is shown via a `XUiC_WindowSelector`).
`XUiC_MapArea.OnOpen` calls `windowManager.Open("windowpaging")` and `OnClose`
calls `windowManager.Close("windowpaging")`. So to close the whole map from your
own button handler: `xui.playerUI.windowManager.Close("windowpaging")`
(`windowManager` is `GUIWindowManager`).

**Two-icon toggle button (start ⇄ cancel).** Put two overlaid `<button>`s at the
same `pos` inside a wrapper `<rect controller="Mod.MyController, Mod">` and have
the controller toggle `child.ViewComponent.IsVisible` so exactly one shows — no
runtime sprite/tooltip mutation needed. Gate the group's own visibility per-button
(don't rely on the parent rect's `IsVisible` cascading). Refresh in
`OnOpen()` (force) and cheaply each `Update(float)` so a state change from
elsewhere (e.g. the console `uncover stop`) keeps the icon in sync. Wire presses in
`Init()` with a `_wired` guard (Init can run twice). See `XUiC_UncoverMapButton`
in the Uncover mod. Controller-name resolution for mod DLLs still needs the
`AppDomain.AssemblyResolve` handler in `InitMod` — see
[XUi - Controllers (C#)](XUi%20-%20Controllers%20(C%23).md).
