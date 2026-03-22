# Prefab Import Pipeline

The full flow from prefab selection to placed voxels in the 3D scene.

**Source**: `index.html` lines ~4413–4762

## Phase 1: Prefab Selection

1. `prefabs-index.json` is loaded at startup (contains name, dir, size, difficulty, yOffset, zoning, tags for each prefab)
2. User opens prefab browser modal with searchable/filterable list
3. User selects a prefab → `selectPrefab(p, el)` is called
4. Details shown: size, difficulty, Y offset, zoning, tags

## Phase 2: File Fetching

```javascript
const basePath = `game-data/prefabs/${p.dir}/${p.name}`
const [ttsResp, nimResp] = await Promise.all([
    fetch(`${basePath}.tts`),
    fetch(`${basePath}.blocks.nim`)
])
```

Both files are downloaded in parallel.

## Phase 3: Data Parsing

```javascript
const tts = parseTTS(ttsBuf)   // → {version, w, h, d, blocks[]}
const nim = parseNIM(nimBuf)   // → Map<blockId → name>
```

See [[TTS File Format]] and [[NIM File Format]].

## Phase 4: Grid Resizing

```javascript
const gridSize = Math.max(tts.w, tts.d)  // Square grid from max horizontal dimension
const yOff = Math.abs(p.yOffset)
resizeGrid(gridSize, yOff)  // Recreates ground plane, grids, camera
```

## Phase 5: Block Placement

Blocks are processed in batches of 200 per frame for UI responsiveness.

For each block in `tts.blocks`:
1. **Resolve name**: `blockName = nim.get(block.blockId)` — see [[Block Definition Resolution]]
2. **Get definition**: `blockDef = blockDefMap.get(blockName)`
3. **Calculate position**:
   ```
   px = block.x + 0.5
   py = block.y - yOff + 0.5
   pz = block.z + 0.5
   ```
   See [[Coordinate System]] for details on the +0.5 centering and Y-offset.
4. **Load geometry**: `loadShapeGeometry(getShapeModelName(blockDef))`
5. **Create mesh** with appropriate materials
6. **Apply rotation**: `applyRotationToMesh(mesh, block.rotation)` — see [[Rotation System]]
7. **Register in scene**: add to `scene`, `placedBlocks`, `occupiedCells`, `blockDataMap`

## Phase 6: Multi-Block Handling

If the block has `multiBlockDim` larger than [1,1,1]:
1. Calculate rotated dimensions using quaternion math — see [[Multi-Block Rotation]]
2. Generate occupancy cells for all affected positions
3. Create invisible occupancy meshes for each cell
4. Link all cells to anchor block via `multiBlockVisual` reference
5. Skip cells that are already occupied (prevents overlap)

## Phase 7: Paint Texture Application

After each block is placed, per-face paint textures from the TTS are applied.

**Key insight**: Paint data in TTS is stored in **block-local** orientation, not world-space.
All instances of the same shape variant have identical paint patterns regardless of rotation.
This means no rotation remapping is needed — just the `MESH_TO_GAME` face-order conversion
from game face indices (Top,Bottom,North,West,South,East) to mesh material indices
(Bottom,East,North,South,Top,West).

For each block with `paintFaces`:
1. Convert game-order paint IDs to mesh-order using `MESH_TO_GAME`
2. If all 6 faces have the same paint → apply single material (fast path)
3. If mixed → build 6-material array, paint for painted faces, base material for unpainted
4. If geometry lacks face groups → auto-generate 6 equal groups

## Error Handling

| Condition | Behavior |
|-----------|----------|
| Missing block definition | Skip block (`continue`) |
| Cell already occupied | Skip block |
| Air block (raw value = 0) | Skip during TTS parsing |
| Unknown rotation index | Default to rotation 0 |
| Shape variant not found | Try base name (split on `:`) |
