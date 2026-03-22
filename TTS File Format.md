# TTS File Format

**TerraTerrain Storage** — binary format storing the 3D voxel grid of a prefab.

**Source**: `index.html` `parseTTS()`

## File Structure

| Section | Size | Description |
|---------|------|-------------|
| Header | 14 bytes | Magic + version + dimensions |
| Block data | total × 4 bytes | uint32 per cell (blockId + rotation) |
| Density | total × 1 byte | uint8 per cell (stability/density) |
| Damage | total × 2 bytes | uint16 per cell (block damage) |
| Texture section | variable | Paint texture overrides (see below) |

Where `total = w × h × d`. All values are **little-endian**.

### Header (14 bytes)

| Offset | Type | Description |
|--------|------|-------------|
| 0 | char[4] | Magic number: `"tts\0"` |
| 4 | uint32 | Version |
| 8 | uint16 | Width (X dimension) |
| 10 | uint16 | Height (Y dimension) |
| 12 | uint16 | Depth (Z dimension) |

## Block Data Encoding

Each block is a 32-bit unsigned integer, bit-packed:

```
Bits  0-14  (15 bits): blockId (0–32767)
Bits 16-20  (5 bits):  rotation index (0–31, only 0–23 used)
```

Extraction:
```javascript
const blockId  = raw & 0x7FFF          // Lower 15 bits
const rotation = (raw >> 16) & 0x1F    // Bits 16-20
```

A raw value of `0` means **air** (empty cell) and is skipped.

## Grid Iteration Order

Blocks are stored in a linear array. The iteration order is **Z-major → Y → X**:

```javascript
for (i = 0; i < w * h * d; i++) {
    z = Math.floor(i / (h * w))
    y = Math.floor((i % (h * w)) / w)
    x = (i % (h * w)) % w
}
```

## Texture/Paint Section

After the block + density + damage layers, paint texture overrides are stored:

1. **SimpleBitStream** — one bit per cell indicating which blocks have paint
   - `int32`: byte length of the bitstream
   - `byte[length]`: bitstream data (bit N = cell N has paint, LSB-first per byte)

2. **Paint data** — 8 bytes per flagged block (in cell order):
   - `byte[6]`: paint ID for each face (Face0–Face5), from `painting.xml` `id` attribute
   - `byte[2]`: padding (always 0)

The paint `id` maps to a `TextureId` property in `painting.xml`, which is the actual texture file to load from `game-data/textures/{TextureId}.png`.

## Face Order (Game Order)

The 6 face paint bytes use the **game's face ordering**, which differs from the OBJ mesh face group ordering:

| Index | Face | Direction |
|-------|------|-----------|
| 0 | Top | Y+ |
| 1 | Bottom | Y- |
| 2 | North | Z- |
| 3 | West | X- |
| 4 | South | Z+ |
| 5 | East | X+ |

This same ordering is used by `blocks.xml` `Texture` property (confirmed by dev comment in XML).

**OBJ mesh face groups** use alphabetical ordering: B=0, E=1, N=2, S=3, T=4, W=5. The import pipeline remaps between these two orderings using `GAME_TO_MESH` / `MESH_TO_GAME` constants.

Paint data is stored in **world space** — face indices refer to absolute directions regardless of block rotation. The import pipeline applies `ROT_LOCAL_TO_WORLD` to convert rotated local faces to world directions before looking up paint.

## Parsed Output

```javascript
{
    version: number,
    w: number, h: number, d: number,
    blocks: [
        { x, y, z, blockId, rotation, cellIndex, paintFaces },
        // paintFaces: [6] array of per-face paint IDs, or null if no paint
        ...
    ]
}
```

See also: [[NIM File Format]] for resolving `blockId` to block names.
