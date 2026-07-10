# Procedural Texture Fills & Main-Thread Stalls

Part of the [7DTD Modding Knowledgebase](README.md). Applies to any mod code
that fills or updates a `Texture2D` procedurally (minimaps, map overviews,
generated icons). Measured July 2026 in the shipped Steam client (Mono).

## The measurement (Uncover minimap)

A 1024×1024 window fill — 4,096 `IMapChunkDatabase.GetMapColors` reads plus ~1M
texel writes of trivial per-texel math — took **~4.5–5s on the main thread**
right after spawn (hard freeze). Two write paths were tried with essentially the
same result:

- `Texture2D.GetRawTextureData<Color32>()` NativeArray indexer: ~4.9s
- managed `Color32[]` + `SetPixels32`: ~4.5s

So the texture WRITE path was **not** the dominant cost (early conclusion here
blamed NativeArray writes — wrong). The same loop off the main thread and/or
with per-section timing points at either the per-chunk DB reads or Boehm-GC
stop-the-world pauses landing inside whatever the main thread happens to be
running during the post-spawn chunk-load allocation storm. (Check the
`[Uncover]` minimap render log line, which splits total vs GetMapColors time,
for the current numbers.)

## The robust fix: render off the main thread

Don't do large texture composition on the main thread at all, regardless of
apparent per-texel cost — during world/chunk load the main thread cannot afford
even "should be fast" mega-loops. Pattern (see `MinimapTexture` in Uncover):

- A dedicated background worker renders into its own `Color32[]` scratch and
  commits into a shared `Color32[]` under a small lock (element tearing during
  a concurrent read = one-frame artifact, acceptable for a minimap).
- The main thread only enqueues coalesced jobs (latest-wins for full renders;
  drop partials while a full is pending) and uploads finished pixels with
  `SetPixels32` + `Apply(false)` (a few ms for 1024² RGBA32).
- Reading the fog DB (`GetMapColors`) from a worker is safe — the DB locks
  internally and vanilla already touches it from its own load thread.
- Guard commits with a generation/origin check so a recenter mid-render
  discards stale output instead of committing to wrong offsets.

`SetPixels32` needs a managed array anyway, so keep the managed backing buffer
as the shared surface; consumers (e.g. an IMGUI compositor) can read it
directly instead of `GetRawTextureData`.

## Diagnosing main-thread stalls generally

A reusable trick when the game freezes with nothing in the log: a MonoBehaviour
that measures the gap between its `Update()` calls (plus `GC.CollectionCount` /
`GC.GetTotalMemory` deltas across the gap), and generic Harmony prefix/postfix
stopwatches (`Stopwatch.GetTimestamp` in `__state`) on suspect methods,
reporting per-method totals whenever a frame gap exceeds 0.25s. Log tell for
"main thread stalled" without any probe: `ChunkManager mesh regeneration thread
blocked/resumed after N.Ns - VML max queued exceeded` — the regen thread's
queue isn't being drained. See `StallProbe.cs` in the Uncover mod.
(`StackTrace(Thread)` capture from a watchdog thread is NOT supported by the
game's Mono — returns nothing.)
