# Structural Integrity & Falling Blocks

Part of the [7DTD Modding Knowledgebase](README.md). How the stability (SI)
system decides blocks fall, and how to move/place floating blocks at runtime
without the world collapsing them. Verified against Assembly-CSharp 3.0.x
(2026-07, Elevator mod).

---

## The stability channel

- Every block cell has a **stability byte 0–15** stored in a chunk channel
  (`chnStability`), persisted with the chunk.
- **15 = full support** (terrain-anchored / bedrock-grade). A neighbor with
  stability 15 counts as infinite support from below.
- **1** = placed-but-weak (what non-`StabilitySupport` blocks get).
- **0** = unsupported → candidate for falling.
- Read/write: `world.GetStability(Vector3i)` / `world.SetStability(Vector3i, byte)`
  (public, no side effects — writes the channel without spreading).
- Block XML flags: `StabilitySupport` (can support neighbors),
  `StabilityIgnore` (excluded from SI entirely),
  `StabilityFull` (placement force-stamps 15 — see below).

## Who computes what (3 actors)

| Actor | When | What |
|---|---|---|
| `StabilityInitializer` ("lighting thread") | chunk generation/decoration | initial full-chunk distribution |
| `ChannelCalculator` (inside `StabilityCalculator`) | **synchronously** during every `SetBlock`/`SetBlocksRPC` when `GameManager.bPhysicsActive` | spreads/unspreads stability around the changed cell (`BlockPlacedAt`/`BlockRemovedAt`) |
| `StabilityCalculator.UpdatePhysics` coroutine | **async**, ~every 0.1 s, time-sliced | drains queues of suspect positions; floods stab-0 regions (`physicsIsolation`) and mass-vs-support checks (`CalcPhysicsStabilityToFall`); any position it deems lost goes to `World.AddFallingBlock` |

Placement stability: `ChunkCluster.SetBlock` calls
`stabilityCalcMainThread.BlockPlacedAt(pos, _bForceFullStabe: bv.Block.StabilityFull)`.
So only blocks with the `StabilityFull` XML property get 15 for free; everything
else derives from the cell below / best neighbor.

## The single choke point: `World.AddFallingBlock`

**Every** fall path funnels through
`World.AddFallingBlock(Vector3i _blockPos, bool includeOversized = false)`:

- `World.AddFallingBlocks(IList<Vector3i>)` just loops into it.
- The SI coroutine calls it for isolated stab-0 clusters and failed
  mass/support checks.
- `EntityAlive` calls it when an entity stands on a block whose stability is 0
  (log signature: `WRN EntityAlive <name> AddFallingBlock stab 0 happens?` —
  if you see this, your runtime-placed blocks have stab 0).

It enqueues into `world.fallingBlocks`; `World.LetBlocksFall()` later converts
queued cells to `EntityFallingBlock` entities (blocks visually drop and break).
A Harmony **prefix returning false** on `AddFallingBlock` is a complete veto
for protected positions (it also skips `ischild`, air, and `StabilityIgnore`
blocks by itself).

## Recipe: moving/holding blocks in mid-air (elevator car pattern)

Two layers, both needed (Elevator mod, `src/ElevatorStability.cs` +
`ElevatorMover.cs`):

1. **After each `SetBlocksRPC` batch, stamp `world.SetStability(pos, 15)` on
   every non-air cell you re-placed.** The synchronous ChannelCalculator has
   already run by the time `SetBlocksRPC` returns, so the stamp sticks. This
   stops the entity-standing check (reads current value) and makes the async
   coroutine treat the volume as supported. **Skip air cells** — stability on
   air lets a block placed there later inherit phantom support
   (`ChannelCalculator.BlockPlacedAt` reads the cell below's stability).
2. **Harmony-prefix `World.AddFallingBlock`** to skip positions inside the
   protected volume. Needed because the coroutine drains positions queued
   *before* your stamp landed (`physicsIsolation` adds its seed position to
   the falling set unconditionally, even if the cell's stability has since
   changed). Protect the volume at rest too, not just mid-move — any nearby
   recalc can re-queue car cells.

Gotchas:

- Stab-15 cells legitimately support neighbors: players can build off the
  floating volume, and when it moves away the vanilla unspread makes those
  attachments fall. Usually the desired physics.
- To block placing/attaching blocks against a moving volume, postfix
  `Block.CanPlaceBlockAt(WorldBase _world, Vector3i _blockPos, BlockValue
  _blockValue, bool _bOmitCollideCheck)` and force `__result = false` inside
  the padded volume (the placement ghost turns red).
- `Chunk.StopStabilityCalculation` exists (per-chunk kill switch, used by the
  prefab editor) but is too coarse for live worlds — it disables SI for
  everything in the chunk.
- `GameManager.bPhysicsActive` is the global SI switch — also too coarse.

## Related

- [Blocks](Blocks.md) — `world.SetBlock` `updatePhysics` param: passing
  `false` leaves placed blocks at stability 1 (same root cause as the
  "stab 0" entity warning).
- [Harmony Patching](Harmony%20Patching.md) — patch patterns used above.
