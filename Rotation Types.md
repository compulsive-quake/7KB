# Rotation Types

Per-block constraints on which of the 24 rotations are allowed.

**Source**: `index.html` lines ~1065–1074

## ROTATION_SETS

| Type | Indices | Count | Description |
|------|---------|-------|-------------|
| `Basic90` | [0, 1, 2, 3] | 4 | Y-axis only (horizontal spin) |
| `Basic45` | [0, 1, 2, 3] | 4 | Same as Basic90 (45° not supported in 3D) |
| `Basic90And45` | [0, 1, 2, 3] | 4 | Same as Basic90 (45° not supported in 3D) |
| `Headfirst` | [0–7] | 8 | Y-axis + upside-down |
| `Sideways` | [0–15] | 16 | Y-axis + upside-down + north/south |
| `No45` | [0–23] | 24 | All 24 orientations |
| `Advanced` | [0–23] | 24 | All 24 orientations |
| `All` | [0–23] | 24 | All 24 orientations |

## Defaults

- **Shape blocks** (blocks with OBJ models): default to all 24 rotations
- **Regular blocks**: default to 4 Y-axis rotations (Basic90)

## Inheritance

Shape variant blocks can inherit rotation type from:
1. Block-level `rotationType` property (highest priority)
2. Parent shape's `rotationType`
3. Default for shape vs. regular block

## Impact

The rotation type determines which indices are available in:
- [[Rotation System#Cycling Rotation (User Interaction)|Rotation cycling]] via Ctrl+scroll
- Validation when placing blocks
- Available options in the rotation UI
