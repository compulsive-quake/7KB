# Block Definition Resolution

How block IDs from prefab files resolve to usable block definitions.

**Source**: `index.html` lines ~3839–3851 (blockDefMap), ~4689–4699 (resolution), ~1353–1480 (XML parsing)

## Resolution Flow

```
TTS blockId → NIM lookup → blockName → blockDefMap → blockDef
```

### Step 1: ID to Name

```javascript
const blockName = nim.get(block.blockId)  // NIM map from .blocks.nim file
```

### Step 2: Name to Definition

```javascript
let blockDef = blockDefMap.get(blockName)
if (!blockDef) {
    // Try base name (remove shape suffix after colon)
    const baseName = blockName.split(':')[0]
    blockDef = blockDefMap.get(baseName)
}
```

### Block Definition Map Construction

The `blockDefMap` combines `allBlocks` and `hiddenBlocks`:

```javascript
for (const b of [...allBlocks, ...hiddenBlocks]) {
    if (b.isShapeVariant) {
        _blockDefMap.set(`${b.name}:${b.shape}`, b)  // e.g., "steelShapes:cube"
    } else {
        _blockDefMap.set(b.name, b)                   // e.g., "woodShapes"
    }
}
```

Shape variants use a `name:shape` composite key. Non-variant blocks use just the name.

## Block Definitions Source

Parsed from `game-data/config/blocks.xml` at startup. Key properties:

| Property | Description |
|----------|-------------|
| `name` | Block identifier |
| `shape` | 3D model shape name |
| `model` | Entity model path (for ModelEntity blocks) |
| `multiBlockDim` | Multi-block dimensions e.g. `[3,2,1]` |
| `allowedRotations` | Rotation constraint type name |
| `texture` | Texture ID(s) |
| `material` | Material name |
| `mapColor` | Map display color |
| `categories` | Category list for UI grouping |
| `isShapeVariant` | Whether derived from shapes.xml |

## Fallback Behavior

If no block definition is found (even after trying the base name), the block is **skipped** during prefab loading.
