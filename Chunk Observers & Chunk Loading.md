# Chunk Observers & Chunk Loading

How 7DTD decides which chunks to load, mesh, and display — and how a mod can
make terrain load around something that isn't the player (camera drone, remote
viewpoint, teleport preload).

## The one API that matters

```csharp
ChunkManager.ChunkObserver obs = world.m_ChunkManager.AddChunkObserver(
    worldPos,                       // absolute world coords (NOT Origin-shifted)
    _bBuildVisualMeshAround: true,  // true = also mesh/display; false = data only
    viewDim,                        // radius in chunks; see caps below
    -1);                            // entityId to stream chunks to; -1 = local/none
obs.SetPosition(newWorldPos);       // every frame; cheap (just stores the vector)
world.m_ChunkManager.RemoveChunkObserver(obs);
```

`GameManager.Instance.AddChunkObserver(...)` / `RemoveChunkObserver(...)` are
thin public wrappers over the same calls.

- Vanilla creates the SP local-player observer exactly like this:
  `AddChunkObserver(pos, true, GameUtils.GetViewDistance(), -1)`.
- For remote clients the server uses `(pos, false, viewDim, player.entityId)` —
  `bBuildVisualMeshAround: false` because the client builds its own meshes.
- On a **client** in MP, a locally added observer is useless for data: chunk
  loading is gated by `ConnectionManager.IsServer` — only server-side observers
  cause chunks to load. Works fine in SP (integrated server).

## viewDim semantics and caps

- `viewDim` is in chunks; the observer covers rings `0 .. viewDim+2` — a
  `(2*(viewDim+2)+1)²`-ish square of *data* chunks (`rectanglesAroundPlayers`).
- Hard engine cap: `rectanglesAroundPlayers` has **15 rings**, so
  `viewDim + 2 <= 15` → viewDim ≤ 13. Vanilla clamps client observers to 12.
  Stay ≤ 12.
- `GameUtils.GetViewDistance()` = `GameStats AllowedViewDistance`, which in SP
  is set from the **video option** `OptionsGfxViewDistance`
  (`GameModeAbstract.Init`). Typically 6–12.
- The outermost ~2 rings of an observer never get a visual mesh: mesh
  regeneration requires the chunk to be lit + decorated, which requires all
  neighbors present. Effective *displayed* radius ≈ viewDim rings.

## The pipeline is 100% observer-driven

Every stage keys off the observer lists — there is **no** "near the player" or
camera-frustum gating anywhere in ChunkManager:

1. `DetermineChunksToLoad` — per observer, recomputed only when it crosses a
   chunk boundary; merges all observers into `m_AllChunkPositions` (data) and
   `m_ViewingChunkPositions` (observers with `bBuildVisualMeshAround`).
2. `GetNextChunkToProvide` — serves *all* observer positions ring-by-ring
   (near rings of every observer first, so a second observer doesn't starve).
3. Lighting queue = `m_AllChunkPositions`; decoration needs neighbors loaded.
4. `RegenerateNextChunk` meshes chunks in `m_ViewingChunkPositions` once lit +
   decorated; `doCopyChunksToUnity` displays them. The ChunkGameObject pool
   grows on demand (`new GameObject("Chunk (new)")`) — no fixed budget.

Practical consequence: a remote observer **does** load and display terrain in
SP, but the generate→light→decorate→mesh→copy latency is seconds per area. A
fast-moving observer (FPV drone at 25+ m/s) outruns it — the detail bubble
trails the flight. Fix: **lead the observer along the velocity vector** (FPV
mod leads by 2 s of horizontal velocity, capped at 64 m) and consider +2
viewDim over the video option since the whole bubble is in frame from altitude.

## Debugging chunk loading

- Vanilla console command `chunkobserver` (alias `co`):
  `co add <x> <z> [radiusChunks]`, `co remove <x> <z>`, `co list`. Places
  **data-only** observers (`bBuildVisualMeshAround: false`) — good for testing
  the load half, not the display half.
- The periodic `Time:` log line decodes as: `Chunks:` = chunks in cache
  (includes chunks held by DynamicMesh etc., not just observers), `CGO:` =
  displayed ChunkGameObjects, `CO:` = observer count (sleeper volumes and
  wandering-horde spawners add temporary ones, so it fluctuates without mods).
- Per-chunk ground truth from a mod: `world.ChunkCache.GetChunkSync(
  WorldChunkCache.MakeChunkKey(cx, cz))` → null = not cached; `.IsDisplayed`
  = has a live visual mesh. All public in the shipped (publicized) assembly.

## Related

- Decompiling for reference: see [[Vehicles]] §"Decompiling vanilla source" —
  `ilspycmd -t ChunkManager "<GameDir>/7DaysToDie_Data/Managed/Assembly-CSharp.dll"`.
- On this machine the game is at `D:\SteamLibrary\steamapps\common\7 Days To Die`
  (NOT the `C:\Program Files (x86)` default in some csproj files — pass
  `-p:GameDir=...` when building outside modman).
