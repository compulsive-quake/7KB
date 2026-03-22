# Block Data Structures

Key data structures used in the prefab import and block placement systems.

## TTS Block Entry

Returned by `parseTTS()` for each non-air block:

```javascript
{
    x: number,        // 0 to w-1 (grid column)
    y: number,        // 0 to h-1 (grid row/height)
    z: number,        // 0 to d-1 (grid depth)
    blockId: number,  // 0–32767 (15 bits from TTS)
    rotation: number  // 0–31 (5 bits from TTS, only 0–23 used)
}
```

## Block Definition

Parsed from `game-data/config/blocks.xml`:

```javascript
{
    name: string,               // "woodShapes", "steelDoor01"
    shape: string,              // "New", "cube", "wedge", etc.
    model: string,              // Entity model path (ModelEntity blocks)
    material: string,           // Material name
    texture: string,            // Texture ID
    color: number,              // Hex color fallback (0xRRGGBB)
    categories: string[],       // ["Shapes", "Building"]
    multiBlockDim: number[],    // [1,1,1] or [3,2,1]
    allowedRotations: number[], // Indices from ROTATION_SETS
    isShapeVariant: boolean,    // true if derived from shapes.xml
    mapColor: string,           // Map.Color property
}
```

## Block Data Map Entry

Stored per placed mesh in `blockDataMap` (keyed by `mesh.uuid`):

```javascript
{
    name: string,                    // Block name
    blockDef: object,                // Full block definition
    rotation: number,                // 0–23 rotation index
    anchorPos: THREE.Vector3,        // Position of block origin
    multiBlockCells: string[],       // Cell keys: ["1,2,3", "2,2,3", ...]
    multiBlockVisual?: THREE.Mesh,   // Visual mesh (multi-blocks)
    isOccupancyCell?: boolean,       // true for invisible occupancy placeholders
    singleBlockVisual?: THREE.Mesh,  // Visual mesh (single blocks)
}
```

## Prefab Index Entry

From `game-data/prefabs-index.json`:

```javascript
{
    name: string,       // "abandoned_house_01"
    dir: string,        // "POIs", "Parts", "RWGTiles", "Test"
    size: { x, y, z },  // Prefab dimensions
    difficulty: number, // 0–5
    zoning: string,     // "ResidentialOld", "none", etc.
    tags: string,       // Comma-separated tag list
    editorGroups: string,
    yOffset: number,    // Negative = underground levels
    hasPreview: boolean
}
```

## Occupied Cells Map

```javascript
occupiedCells: Map<string, string>
// Key: cell key "x,y,z" (floored integers)
// Value: mesh UUID of the block occupying that cell
```

Prevents placing two blocks at the same grid position.
