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
