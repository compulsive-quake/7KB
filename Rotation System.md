# Rotation System

7 Days to Die uses 24 discrete orientations for blocks, corresponding to the 24 orientation-preserving symmetries of a cube (6 faces × 4 rotations per face).

**Source**: `index.html` lines ~1045–1100

## ROTATIONS_24 Lookup Table

Each rotation index maps to Euler angles `[rotX, rotY, rotZ]` in radians:

| Index | Face | Y-Spin | Euler [X, Y, Z] |
|-------|------|--------|------------------|
| 0 | Up | 0° | `[0, 0, 0]` |
| 1 | Up | 90° | `[0, π/2, 0]` |
| 2 | Up | 180° | `[0, π, 0]` |
| 3 | Up | 270° | `[0, -π/2, 0]` |
| 4 | Down | 0° | `[π, 0, 0]` |
| 5 | Down | 90° | `[π, -π/2, 0]` |
| 6 | Down | 180° | `[π, π, 0]` |
| 7 | Down | 270° | `[π, π/2, 0]` |
| 8 | North | 0° | `[π/2, 0, 0]` |
| 9 | North | 90° | `[π/2, π/2, 0]` |
| 10 | North | 180° | `[π/2, π, 0]` |
| 11 | North | 270° | `[π/2, -π/2, 0]` |
| 12 | South | 0° | `[-π/2, 0, 0]` |
| 13 | South | 90° | `[-π/2, -π/2, 0]` |
| 14 | South | 180° | `[-π/2, π, 0]` |
| 15 | South | 270° | `[-π/2, π/2, 0]` |
| 16 | East | 0° | `[0, 0, -π/2]` |
| 17 | East | 90° | `[π/2, 0, -π/2]` |
| 18 | East | 180° | `[π, 0, -π/2]` |
| 19 | East | 270° | `[-π/2, 0, -π/2]` |
| 20 | West | 0° | `[0, 0, π/2]` |
| 21 | West | 90° | `[-π/2, 0, π/2]` |
| 22 | West | 180° | `[π, 0, π/2]` |
| 23 | West | 270° | `[π/2, 0, π/2]` |

### Pattern

- Groups of 4: each face direction has 4 Y-axis spin variants (0°, 90°, 180°, 270°)
- **Face Up/Down**: rotate around X by 0 or π, spin around Y
- **North/South**: rotate around X by ±π/2, spin around Y
- **East/West**: rotate around Z by ∓π/2, spin around X

## Applying Rotation to a Mesh

```javascript
function applyRotationToMesh(mesh, rotIndex) {
    const r = ROTATIONS_24[rotIndex] || ROTATIONS_24[0];
    mesh.rotation.set(r[0], r[1], r[2]);
}
```

Falls back to rotation 0 (identity) if the index is out of range.

## Rotation Labels

```javascript
const FACE_NAMES = ['Up','Up','Up','Up', 'Down','Down','Down','Down',
    'North','North','North','North', 'South','South','South','South',
    'East','East','East','East', 'West','West','West','West'];

function getRotationLabel(rotIndex) {
    const face = FACE_NAMES[rotIndex];
    const yRot = (rotIndex % 4) * 90;
    return `${face} ${yRot}°`;  // e.g., "North 90°"
}
```

## Binary Encoding

In [[TTS File Format]], rotation is stored as 5 bits (bits 16–20) of the block's uint32 value:
```javascript
const rotation = (raw >> 16) & 0x1F;
```

## Cycling Rotation (User Interaction)

```javascript
function cycleRotation(currentRot, allowedRots, forward) {
    const idx = allowedRots.indexOf(currentRot);
    if (idx === -1) return allowedRots[0];
    if (forward)
        return allowedRots[(idx + 1) % allowedRots.length];
    else
        return allowedRots[(idx + allowedRots.length - 1) % allowedRots.length];
}
```

- **Ctrl+scroll wheel** cycles through allowed rotations
- Wraps around at both ends
- See [[Rotation Types]] for allowed rotation sets

## Related

- [[Rotation Types]] — per-block rotation constraints
- [[Multi-Block Rotation]] — how rotation affects multi-block dimensions
- [[X-Mirror Rotation]] — rotation correction for mirrored prefabs
