# Moving Blocks at Runtime (SetBlocksRPC) & Rendering Blocks as Moving Meshes

How to relocate world blocks in code (elevators, moving platforms) and how to
render an arbitrary `BlockValue` as a free-gliding Unity GameObject for smooth
motion. Verified against the 3.0.x Assembly-CSharp; working implementation in
the Elevator mod (`src/ElevatorMover.cs`).

## Teleporting blocks (whole-block steps)

Voxel blocks only exist at integer grid positions — "moving" a block means
remove + re-place. Batch everything through one `World.SetBlocksRPC(List<BlockChangeInfo>)`:

1. **Capture** each non-air, non-child cell: `BlockValue` (rotation/damage are
   inside), `world.GetDensity(pos)`, `world.ChunkCache.GetTextureFullArray(pos)`
   (paint), `world.GetTileEntity(pos)`.
2. **Detach tile entities first** so contents survive:
   `te.GetChunk().RemoveTileEntity(world, te)`.
3. One batch: `new BlockChangeInfo(new BlockValueRef(src), BlockValue.Air)` for
   every source, then `new BlockChangeInfo(new BlockValueRef(dst), bv, density, tex)`
   for every destination.
4. **Reattach TEs**: get destination chunk via `world.GetChunkSync(World.toChunkXZ(x), World.toChunkXZ(z))`,
   `chunk.RemoveTileEntityAt<TileEntity>(world, World.toBlock(dst))`, set
   `te.localChunkPos = World.toBlock(dst)`, `chunk.AddTileEntity(te)`.
5. Only capture/write **master** blocks — multiblock children (`bv.ischild`)
   are created/removed automatically with their master.
6. Runtime-placed blocks float unsupported → see
   [Structural Integrity & Falling Blocks](Structural%20Integrity%20%26%20Falling%20Blocks.md)
   for the stability-15 stamp + `AddFallingBlock` veto recipe.

## Smooth motion: proxy mesh while blocks are "in transit"

Whole-block steps visibly pop. For a smooth glide: **remove the real blocks,
render a proxy GameObject, re-place the blocks at the destination**. The proxy
uses the exact path vanilla `EntityFallingBlock.CreateMesh` uses:

```csharp
// One container GO per block cell, pivot at the BLOCK CENTER (+0.5 each axis).
// Pass the REAL BlockValue: rotation, damage state and paint are baked into
// the generated mesh (BlockShapeNew.renderFace reads _blockValue.rotation).
ItemClassBlock.CreateMesh(null, world, bv, null, worldPos /*light sampling*/,
    cell.transform, BlockShape.MeshPurpose.Local, textureFullArray);

// Colliders so entities can stand on the proxy — but NOT for
// BlockShapeModelEntity (its CloneModel branch ships real colliders and
// ignores the purpose, so a second call would duplicate the whole model):
if (!(bv.Block.shape is BlockShapeModelEntity))
    ItemClassBlock.CreateMesh(null, world, bv, null, worldPos,
        cell.transform, BlockShape.MeshPurpose.SimplifiedCollisionOnly, textureFullArray);

Utils.SetColliderLayerRecursively(root, 16); // "TerrainCollision": real ground to every mover.
    // (13 "Items" = vanilla falling blocks is hidden from block targeting, but NO character
    // controller collides with it — see Moving Platforms & Riding Entities)
```

Gotchas learned the hard way:

- `EntityFallingBlock` calls `CloneModel(world, bv.ToItemValue(), ...)` and then
  re-applies `shape.GetRotation(bv)` on the transform — because **`ToItemValue()`
  strips rotation**. Calling `ItemClassBlock.CreateMesh` with the real
  `BlockValue` directly skips that dance and gets exact geometry (incl. 45°
  rotations and model-entity offsets). Do NOT also rotate the transform.
- `MeshPurpose.Local` affects lighting/UV mode only, not rotation.
- The mesh GO gets `UpdateLightOnChunkMesh` added automatically (world light
  follows the proxy).
- 7DTD shifts the rendered world by `Origin.position` — set
  `root.transform.position = worldSpacePos - Origin.position` every frame and
  origin shifts are handled for free.
- Put a kinematic `Rigidbody` on the proxy root (moving colliders) and set
  `transform.position` every **render** frame — anything advancing at
  fixed-step rate (MovePosition/FixedUpdate, UFPS `m_Platform`) stair-steps
  ~5 cm per physics tick and looks shaky/blurry.
- Don't blanket-`SetPosition` riders every frame — it stomps player input
  (resets `vp_FPController` smoothing). See
  [Moving Platforms & Riding Entities](Moving%20Platforms%20%26%20Riding%20Entities.md)
  for the full recipe (state-preserving controller shift for the local
  player, suspension physics for driven vehicles, filtered vertical-delta
  carry for the rest).
- Terrain blocks (`shape.IsTerrain()`) render nothing through this path —
  vanilla uses a debris prefab for those.
- **Persistence caveat**: while gliding, the blocks exist only in memory. Land
  them (re-place at the nearest whole-block position) on any abort/world
  shutdown; a hard crash mid-ride loses them unless you persist the capture.
- Pre-check the whole path is clear *before* removing anything, and block
  players from building into the swept volume while riding
  (`Block.CanPlaceBlockAt` postfix).
# Moving Blocks at Runtime (SetBlocksRPC)

Part of the [7DTD Modding Knowledgebase](README.md). How to relocate a volume of
placed blocks (an elevator car, a moving platform) preserving rotation, damage,
density, paint and tile-entity contents. Learned building the Elevator mod
(2026-07, 7DTD 3.0.x). API facts are reflection-verified against Assembly-CSharp.

## The primitive: capture → clear → re-place in one batch

`World.SetBlocksRPC(List<BlockChangeInfo>)` applies changes **in list order** and
handles network sync + chunk updates. To shift a box of blocks by one cell:

```csharp
// 1. Capture every non-air MASTER block in the box (skip bv.ischild —
//    multiblock children are re-created automatically when the master is set).
BlockValue bv = world.GetBlock(pos.x, pos.y, pos.z);
sbyte density = world.GetDensity(pos);
TextureFullArray tex = world.ChunkCache.GetTextureFullArray(pos);   // paint, all faces/channels
TileEntity te = world.GetTileEntity(pos);

// 2. One batch: ALL clears first, then ALL sets — ordering makes overlap safe.
changes.Add(new BlockChangeInfo(new BlockValueRef(srcPos), BlockValue.Air));
changes.Add(new BlockChangeInfo(new BlockValueRef(dstPos), bv, density, tex)); // 4-arg ctor keeps density+paint
world.SetBlocksRPC(changes);
```

- Copying the whole `BlockValue` preserves rotation, damage and meta for free.
- The `BlockChangeInfo(BlockValueRef, BlockValue, sbyte, TextureFullArray)` ctor
  carries density and paint in the same change — no separate SetTextureFull pass.

## The remesh gap: re-placed blocks are invisible & non-solid for frames

`SetBlocksRPC` only writes chunk **data** and flags the touched chunks for
remeshing (`Chunk.NeedsRegeneration`, set synchronously inside the call). The
render mesh and the chunk MeshCollider rebuild asynchronously afterwards:

1. worker thread remeshes the chunk (`InProgressRegeneration` true while it
   runs; leaves generated mesh layers pending → `NeedsCopying` true);
2. main thread copies the mesh into the ChunkGameObject over the following
   frames (`InProgressCopying`; on completion sets `IsCollisionMeshGenerated`).

Until step 2 finishes, re-placed blocks neither render nor collide. If a
stand-in (proxy platform, moving car) is destroyed in the same frame as the
batch, the volume flashes invisible and fast-falling entities tunnel through
it (observed in the Elevator mod: player fell through the car floor at
arrival).

Fix pattern (Elevator's post-landing "linger", `ElevatorMover`): keep the
stand-in GameObject (mesh + colliders) alive after the batch and poll every
chunk under the volume each frame; destroy the stand-in only when every chunk
reports

```csharp
!chunk.NeedsRegeneration && !chunk.InProgressRegeneration &&
!chunk.NeedsCopying && !chunk.InProgressCopying
```

plus a ~3 s timeout backstop (chunk unloaded mid-poll, or kept dirty by
unrelated edits). All four members are public(ized) on `Chunk`; fetch chunks
via `world.GetChunkSync(chunkX, chunkZ) as Chunk`. Unrelated pending edits can
hold the flags true a little longer — harmless, the stand-in just lingers a
few extra frames over identical geometry.

## Proxy visuals: don't use MeshPurpose.Local for painted blocks

`ItemClassBlock.CreateMesh(..., MeshPurpose.Local, tex)` (the falling-block
visual path) force-disables **global UVs** (`BlockShapeNew.renderFace`:
`flag = flag && _purpose != MeshPurpose.Local`). Painted multi-block textures
(most paints: `blockW/blockH` 2 or 4) collapse to a fixed corner sub-tile with
per-face mirroring — a moving proxy visibly shows every painted face "rotated
wrong" until the real chunk mesh takes over again.

Fix: render voxel-block visuals yourself with **`MeshPurpose.World`** — the
exact path the chunk mesher uses. Sub-tiles come from
`ComputeTaIndexOffsFor(...uvPos...)` where `uvPos` ends up the block world
position (floored), so passing the block's world pos as `renderFull`'s first
arg and `(-0.5,-0.5,-0.5)` as draw pos reproduces chunk paint exactly:

```csharp
world.GetSunAndBlockColors(blockPos, out byte sun, out byte blk);
var lighting = new LightingAround(sun, blk, 0);
int mi = bv.Block.MeshIndex;
var meshes = new VoxelMesh[MeshDescription.meshes.Length];
meshes[mi] = VoxelMesh.Create(mi, MeshDescription.meshes[mi].meshType, 1);
if (mi == 0)  // transparent companion layer (glass) — CreateMesh does the same
    meshes[2] = VoxelMesh.Create(2, MeshDescription.meshes[2].meshType, 1);
bv.Block.shape.renderFull(blockPos, bv, new Vector3(-0.5f,-0.5f,-0.5f), null,
    lighting, tex, meshes, BlockShape.MeshPurpose.World);
// then per non-empty mesh:
// VoxelMesh.CreateMeshFilter(i, 0, go, "Item", false, out mf, out mr) + CopyToMesh
```

Keep `MeshPurpose.SimplifiedCollisionOnly` via `CreateMesh` for the collider,
and keep `CreateMesh(Local)` for `BlockShapeModelEntity` blocks (prefab clone,
purpose is ignored there). See `ElevatorMover.BuildCellVisual`.

## Global-UV textures are world-pinned in the SHADER — moving meshes slide

A `MeshPurpose.World` bake is only half the story. For global-UV faces
(`UVRectTiling.bGlobalUV` / `Block.UVMode.Global` — most full-block wall
paints) `BlockShapeNew.renderFace` bakes **no usable mesh UVs**. It stamps
vertex color `a = 1`, `g` = texture-array slice, and packs per-axis **tiling
divisors** into uv0 (`AppendSubsetTriPlanar`: `uv0 = (blockW/xTiling,
blockH/yTiling)`); the opaque chunk shader (**`Game/Overlayed_TA`**, asset
`Assets/Shaders/CustomStandardWithOverlay_TA.shader`, resolved via the
BlockVT `prefabMeshDescriptionCollection`) then computes the texture
coordinate per FRAGMENT from the interpolated **world position**
(`unity_ObjectToWorld * vertex`), triplanar-blended by the world normal:

- X faces: `U = sign(n.x)·wp.z/uv0.x`, `V = wp.y/uv0.y`
- Y faces: `U = sign(n.y)·wp.x/uv0.x`, `V = wp.z/uv0.y`
- Z faces: `U = −sign(n.z)·wp.x/uv0.x`, `V = wp.y/uv0.y`

with `slice = color.g` and REPEAT wrap on the unbounded coords. When
`color.a < 0.5` the shader instead samples `_MainTex[color.g]` at uv0
directly (the mesh-UV path). Static chunks never notice the world pinning —
but a gliding proxy mesh moves under it: walls visibly "slide" through
their texture while the car rides.

**Dead ends, verified 2026-07 (V2.x):** the shader does NOT read the
`_OriginPos` global — overriding it per renderer via MaterialPropertyBlock
does nothing here. Vanilla needs no origin correction because
`Origin.DoReposition` snaps to 16-block multiples (`& -16`) and every block
tiling (1/2/4) divides 16, so shifts are invisible to the periodic mapping.
There is no material-level lever at all: raster position and UV world
position come from the same `unity_ObjectToWorld`.

**Working fix (Elevator, `ElevatorMover.AnchorGlobalUvs`):** after the
`MeshPurpose.World` bake, rewrite every flagged vertex of the texture-array
mesh (`MeshDescription.meshes[i].bTextureArray` — mesh 0 only): compute the
UV the shader would produce at the departure position using the formulas
above (`wp` = mesh vertex + block center − `Origin.position`; plane from the
vertex normal's dominant axis; divisors read from the baked uv0), write it
to uv0 and set `color.a = 0`. Same slice, same texel, sampled through the
mesh-UV path — pixel-identical at liftoff and the paint rides with the mesh.
Sloped global-UV faces lose only the triplanar corner blend. Glass is safe:
`Hidden/Game/ChunkTransparent` (mesh 2) has no world-UV branch.

To inspect compiled shaders: UnityPy reads the `Shader` asset; decompress
`compressedBlob` segments (lz4) per `offsets/compressedLengths/
decompressedLengths` and the GLSL variants are plain text inside — see
`Elevator/.claude/tmp/blob_programs.py`. Addressables GUID → asset path via
`aa/catalog.json` (`m_KeyDataString`/`m_BucketDataString`/`m_EntryDataString`
base64 decode — see `catalog_lookup.py`).

## Chunk-mesh vertex colors are NOT Lighting.ToColor — and proxy light is frozen

**The channel layout (BlockShapeNew, i.e. every modern shape, both texture-
array and not — see `BlockShapeNew.renderFull` ~line 1290):**

- `r` = **sun light**, trilinearly interpolated between the **8 corner values**
  the mesher computes — this is the window-light pattern on walls/floors;
- `g` = **texture-array index** (paint!);
- `b` = stability (SI debug tint);
- `a` = a shader flag.

Only `r` is light. Rewriting `g`/`b`/`a` corrupts textures. The classic
`Lighting.ToColor(sun, block, side)` encoding (`a` = block light) only applies
to legacy shapes (billboards/terrain/water). Block light isn't in vertex
colors at all on the New path — `LightLOD` Unity point lights render it. The
`r` sun values are **time-of-day independent** (sky access; the shader applies
day/night), so a brightness ratio works day and night.

**Corner lighting** (`MeshGenerator.CreateMesh` + `maxLight` + `Lighting3DArray`):
per corner = *average of the non-zero SUN values* of its 2×2×2 cell group
(solid cells read ≈0 and drop out — that's how light pools form under
windows); middle = the cell's own value. `Lighting3DArray` reads SUN only.
This is replicable from `GetSunAndBlockColors` (0–15) — see
`ElevatorMover.CaptureLightingAround`.

Vanilla's `UpdateLightOnChunkMesh` (added by `CreateMesh`/`EntityFallingBlocks`)
is **dead code**: `checkLight()` no-ops unless `SetChunkMesh(vm)` was called,
and no caller in the entire 3.0.x assembly ever calls it (full ilspycmd
decompile). Don't resurrect it: its `UpdateColors` writes uniform
`ToColor` over every vertex — on a texture-array mesh that stomps `g` and
scrambles the block's paint.

Working recipe (Elevator, `ElevatorProxyLight` + `ElevatorMover`):

- **Capture per-cell `LightingAround` BEFORE the removal batch** (light data
  intact) and bake the proxy with it → the parked pattern rides along.
- In flight, scale **only `r`**: per cell, ratio = (probe now + 0.5) /
  (probe at capture + 0.5), probe = max sun over the 3×3×3 neighborhood of
  the moving cell, eased per frame.
- **Gate sampling after liftoff**: the removed cells keep their stale (solid)
  light until the async recalc runs — sampling immediately fades the car to
  black at departure. Hold the gate ~1.2 s or until the car moved ≥2 blocks;
  until then the departure bake is exactly what the rider saw anyway.
- **Async-copy guard**: `CopyToMesh` uploads through `MeshDataManager`
  batches; `mf.sharedMesh` is null for a few frames — don't touch `mf.mesh`
  until `sharedMesh != null` and the vertex count matches.
- **Freeze on landing**: the light data shifts again while the destination
  chunks rebuild; keep the last in-flight ratio until the chunk mesh takes
  over.

## Proxy clones of light blocks ignore on/off state (LightLOD)

A `BlockShapeModelEntity` clone made through `ItemClassBlock.CreateMesh` comes
up in the **prefab's default lit state** — an off ceiling light turns ON while
riding a proxy and snaps back off on landing. Cause: `LightLOD` only applies
the block's real state in `CheckInitialBlock()`, and only when it has a
`BlockEntityData` (`SetBlockEntityData`, called by the chunk display path).
On a hand-made clone `bed == null`, so the branch that reads `meta & 2` and
calls `SwitchOnOff` never runs.

Fix (verified in the Elevator mod, 3.0.x): read the live entity's state
before the block is removed, stamp it on the clone —

```csharp
// capture, BEFORE SetBlocksRPC clears the block:
Chunk chunk = world.GetChunkFromWorldPos(pos) as Chunk;
BlockEntityData bed = chunk.GetBlockEntity(pos);            // world pos
LightLOD[] lods = bed.transform.GetComponentsInChildren<LightLOD>(true);
bool[] states = ...each lods[i].bSwitchedOn...;             // ground truth,
// whoever set it (BlockLight meta bit 2, power grid, modded toggle)

// apply, on the freshly created clone:
cloneLods[i].SwitchOnOff(states[i], _ignoreToggle: true);
```

`SwitchOnOff` just sets `bSwitchedOn`; `LightLOD.FrameUpdate` (driven by
`GameLightManager`, registered in `OnEnable` on instantiate) applies it every
frame to the `Light`, emissive materials (`SetEmissiveColor`), the
`LitRootObject` (flames/fixtures) and audio — so one call covers all visuals.
Same prefab both sides → `GetComponentsInChildren` order matches for
per-index mapping. See `ElevatorMover.CaptureLightStates`/`ApplyLightStates`.

## Proxy clones & re-placed blocks flash the green placement arrow

Many door/panel/trap prefabs ship a `trap_indicatorPrefab` child (green
`trap_arrow_indicator` mesh) that is **ACTIVE in the prefab**; its
`TemporaryObject` script (public, `life = 2f`) shows it for ~2 s after every
instantiation, then `Destroy()`s it — that's the vanilla "which way is it
facing" placement affordance. Consequence for movers: **every fresh
instantiation flashes the arrow**, i.e. the proxy clone at liftoff
(`CloneModel` clones all children) *and* the chunk re-display after the
landing batch (new `BlockEntityData`). Verified carriers (V2.x `doors.bundle`,
via UnityPy dump): `elevatorInsidePanelPrefab`, `elevatorOutsideButtonPanelPrefab`,
`elevatorOutsideFloorPanelPrefab`, hatches, roll-up doors/gates, garage doors,
`TrailerDoorPrefab`.

Fix (Elevator, `ElevatorMover.StripPlacementIndicators` +
`Patch_TemporaryObject_Start`): after any manual `CreateMesh`/`CloneModel`,
`GetComponentsInChildren<TemporaryObject>(true)` → `SetActive(false)` (kills
same-frame rendering; `Destroy` only takes effect at end of frame) +
`Destroy(gameObject)`. For the landing side, a per-frame sweep of
`chunk.GetBlockEntity(worldPos)` **races the chunk display and still flashes
for a frame or two** — the race-free hook is a Harmony **prefix on
`TemporaryObject.Start`** (Start always runs before the object's first
render): destroy + skip original when the object's name contains "indicator"
and its position is inside the just-landed volume within a few seconds of
the landing, so player placements elsewhere keep their arrow.

Inspecting bundle prefab hierarchies without Unity: `UnityPy` (pip, installed
on this machine) reads `Data/Addressables/Standalone/**/*.bundle` directly —
walk `Transform.m_Children`/`m_GameObject`, read `MonoBehaviour.m_Script` for
component class names. See `Elevator/.claude/tmp/dump_prefab.py`.

## Tile entities (chests keep their loot)

Setting a block to air removes its TileEntity **without dropping contents** — a
moved chest would come back empty. Detach/reattach around the swap:

```csharp
// BEFORE the batch: detach so the air-set finds no TE to destroy
te.GetChunk().RemoveTileEntity(world, te);

// AFTER the batch: kill any auto-created TE at the destination, move the old one in
Chunk chunk = world.GetChunkSync(World.toChunkXZ(dst.x), World.toChunkXZ(dst.z)) as Chunk;
Vector3i localPos = World.toBlock(dst);
chunk.RemoveTileEntityAt<TileEntity>(world, localPos);   // generic in 3.0
te.localChunkPos = localPos;
chunk.AddTileEntity(te);
```

## Multiblocks (e.g. the 1×2×1 elevator panels)

- Capture/clear/set only the **master** (`!bv.ischild`); the engine removes and
  re-creates child cells through `OnBlockRemoved`/`OnBlockAdded` (notify runs
  inside `SetBlocksRPC`).
- Collision-checking the path: a destination cell holding a **child whose master
  is inside the moving box** is passable — find the master with
  `destBv.Block.multiBlockPos.GetParentPos(dest, destBv)`.

## Carrying riders

```csharp
Bounds bb = new Bounds();                       // world-space, entity positions are world coords
bb.SetMinMax(minVec, maxVecPlusHeadroom);       // extend ~1.6 above the box top for feet-on-top entities
List<Entity> riders = world.GetEntitiesInBounds((Entity)null, bb, true);
rider.SetPosition(rider.position + new Vector3(0, dy, 0), true);
rider.fallDistance = 0f;                        // avoid accumulating fall damage on down-steps
```

## Instant moves (teleport, no glide) — reuse the ride machinery, not its rider checks

Learned adding jump-to-floor / live height editing to the Elevator settings
window (2026-07). A whole-volume teleport is capture → clear → re-place in a
single frame; `ElevatorMover.Jump` runs the ride path's own `Lift()` +
`Land()` back-to-back (all validation, TE moves, stability stamps, key remaps
for free), then moves the proxy GameObject to the destination and hands it to
the normal linger poll — so the car is visible/solid at the target instantly
while the chunks remesh.

- **Collider-based rider detection cannot work same-frame**: colliders created
  this frame aren't seen by `m_GroundHit` (stale from the last fixed update —
  it still points at the old chunk collider, which outlives the removal batch
  by a few frames anyway), and a physics query may not see them either. Select
  riders **positionally** instead: every entity inside the bounds volume plus
  a ~1.6 head/roof band, no downward padding (feet positions sit ON the floor
  block top, and a -1 pad would wrongly grab entities beneath the car).
- Teleport the local player by shifting `vp_FPController.transform.position`
  and its cached `m_SmoothPosition`/`m_FixedPosition`/`m_PrevPos` in one piece
  (grounded or not — a discrete jump must never leave the camera behind);
  everything else via `SetPosition`, including *driven* vehicles — no solver
  can push a rigidbody through a teleport.
- Doors: don't run the ride's close-everything → open-arrival pair in one
  frame — an open interior (cab) door would get `SetOpen(false)` +
  `SetOpen(true)` back-to-back, firing both sounds at once. Cab doors ride
  the car with their TE state captured, so skip them on close and let the
  arrival open be a no-op if already open.
- Editing-UI pattern: when data edits (floor height) must move blocks to stay
  true, apply the data change, attempt the jump, and **roll the data back if
  the jump refuses** (blocked path, unloaded area) — never let stored geometry
  and world blocks separate.

## Rotating a moved volume (90° steps)

Learned building the AnyGate mod (2026-07). A box rotated in 90° steps stays
**axis-aligned** — dimensions just permute — so a rotated landing volume is
still a simple AABB and all the capture/clear/set machinery above applies.
Represent the whole-structure orientation as an index into the game's own
rotation table (public: `BlockShapeNew.rotationsToQuats`, 28 entries used,
0–23 are the 90° set; `BlockShapeNew.GetRotationStatic(i)` returns the
quaternion — see [[Rotation System]]).

- **Rotated size**: `|q * (sx,sy,sz)|` per component, rounded.
- **Cell mapping** (rotate about the box centre, re-anchor at min corner):
  `q * (cell + 0.5 - size/2) + newSize/2 - 0.5`, floored with a small epsilon
  (values are exact half-integers ± float error; `Mathf.RoundToInt` banker's
  rounding will jitter on .5).
- **Per-block rotation**: compose `target = deltaQ *
  GetRotationStatic(bv.rotation)`, then pick the table index with max
  `|Quaternion.Dot|` **among indices the block supports** —
  `Block.SupportsRotation(byte i)` is public and encodes: i<4 needs Basic90,
  i<8 Headfirst, i<24 Sideways, 24–27 Basic45 (`Block.AllowedRotations`
  flags). Plain non-shape blocks are Basic90-only, so a tipped-over gate
  clamps them to nearest yaw — positions stay exact, only the visual spin
  clamps. Assign via `bv.rotation = (byte)best` before the
  `BlockChangeInfo`; multiblock children re-create themselves at the new
  rotation through `OnBlockAdded`.
- **Composition/inverse of orientations**: multiply the quaternions and
  nearest-match back to an index (the 24 set is closed under composition).
- **Proxy**: parent the per-cell meshes relative to the source box **centre**
  (not min corner) so the whole GameObject can `Slerp(identity, deltaQ, t)`
  while its position lerps between the two volume centres.
- Paint (`TextureFullArray`) is per-face; carrying it unchanged puts painted
  faces on different world sides after rotation. No remap implemented yet.

## Deleting blocks outright (dev-style, no drops)

To delete blocks the way the dev tools do (`BlockTools.CubeRPC` / world-editor
selection delete), batch air **with air density** — plain `BlockValue.Air`
without a density change leaves solid terrain density behind (invisible
collision / terrain mesh remnants when deleting dirt or stone):

```csharp
changes.Add(new BlockChangeInfo(new BlockValueRef(pos), BlockValue.Air, MarchingCubes.DensityAir));
world.SetBlocksRPC(changes);
```

- Delete the **master** position of a multiblock; `MultiBlockArray.RemoveChilds`
  runs from `OnBlockRemoved` and clears the child cells automatically.
- Tile entities on deleted blocks are removed by the normal removal path — no
  manual detach needed (contents are destroyed, which is what deletion wants).

## Other gotchas

- Step the move (e.g. 1 block / 0.4 s from a `ModEvents.UnityUpdate` timer) rather
  than teleporting N blocks — feels like an elevator and keeps each batch small.
- Check `world.IsChunkAreaLoaded(...)` at the box corners before each step and
  clamp `y` to [1, 254].
- Persist the car's **current offset** (not just "current floor") after every
  step, so a crash mid-ride can't desync saved data from where the blocks
  physically are.
- If a control block (button panel) rides inside the moving box and your mod keys
  data by its position, remap the key every step.
