# Multi-Block Rotation

How rotation affects blocks that span multiple grid cells.

**Source**: `index.html` lines ~1126–1209

## Rotated Dimensions

When a multi-block is rotated, its bounding box dimensions change. A 3×2×1 block rotated 90° around Y becomes 1×2×3.

### `getRotatedDims(dims, rotIndex)`

```javascript
function getRotatedDims(dims, rotIndex) {
    if (!dims || (dims[0] === 1 && dims[1] === 1 && dims[2] === 1)) return [1, 1, 1];
    const [W, H, D] = dims;
    const r = ROTATIONS_24[rotIndex] || ROTATIONS_24[0];
    const euler = new THREE.Euler(r[0], r[1], r[2]);
    const q = new THREE.Quaternion().setFromEuler(euler);

    // Project each axis through the rotation
    const vx = new THREE.Vector3(W, 0, 0).applyQuaternion(q);
    const vy = new THREE.Vector3(0, H, 0).applyQuaternion(q);
    const vz = new THREE.Vector3(0, 0, D).applyQuaternion(q);

    return [
        Math.round(Math.abs(vx.x) + Math.abs(vy.x) + Math.abs(vz.x)),
        Math.round(Math.abs(vx.y) + Math.abs(vy.y) + Math.abs(vz.y)),
        Math.round(Math.abs(vx.z) + Math.abs(vy.z) + Math.abs(vz.z)),
    ];
}
```

Uses quaternion rotation to project the original dimension vectors, then sums absolute components to get the axis-aligned bounding box.

## Mesh Positioning

### `computeMultiBlockMeshPos(anchorPos, dims, rotIndex, isEntityGeo, geoBBox)`

For rotated multi-blocks, the visual mesh must be positioned so its geometry aligns with the grid:

1. Get geometry bounding box corners
2. Rotate all 8 corners via quaternion
3. Find the minimum corner in world space
4. Position mesh so geometry min aligns with anchor cell min:
   ```
   meshPos = anchorPos - (0.5, 0.5, 0.5) - worldMin
   ```

## Occupancy Cells

During prefab loading, multi-blocks occupy multiple cells:
1. Rotated dimensions are calculated
2. For each cell in the rotated bounding box, an invisible occupancy mesh is created
3. All occupancy cells are linked to the anchor block via `multiBlockVisual`
4. The `occupiedCells` map prevents duplicate blocks at the same position

## Example

A door with `multiBlockDim: [1, 2, 1]` (1 wide, 2 tall, 1 deep):
- Rotation 0 (Up 0°): occupies (0,0,0) and (0,1,0)
- Rotation 8 (North 0°): rotated dimensions become [1, 1, 2], occupies (0,0,0) and (0,0,1)

## Related

- [[Rotation System]] — the 24-rotation lookup table
- [[Coordinate System]] — cell centering and position calculations
