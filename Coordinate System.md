# Coordinate System

How positions are represented and transformed between file formats and the 3D scene.

## Grid Iteration Order (TTS)

Blocks in the [[TTS File Format]] are stored linearly. Decoded as Z-major → Y → X:

```
z = floor(i / (h × w))           // Outermost loop
y = floor((i mod (h × w)) / w)   // Middle loop
x = (i mod (h × w)) mod w        // Innermost loop
```

## Cell Centering

TTS grid positions are integer cell indices (0-based). The 3D scene uses center-of-cell positioning by adding 0.5:

```
scene_x = tts_x + 0.5
scene_y = tts_y - yOffset + 0.5
scene_z = tts_z + 0.5
```

The constant `HALF = 0.5` is used throughout the codebase.

## Y-Offset

Prefab metadata includes a `yOffset` value (typically negative) indicating how many levels of the prefab are underground:

- `yOffset: 0` — entirely above ground
- `yOffset: -5` — 5 levels below ground level
- `yOffset: -13` — 13 levels below ground level

Applied during placement:
```
scene_y = block.y - Math.abs(yOffset) + 0.5
```

This shifts the entire prefab down so the correct levels end up underground.

## Cell Key System

Each voxel's occupancy is tracked with a string key:

```javascript
cellKey = `${Math.floor(x)},${Math.floor(y)},${Math.floor(z)}`
```

Stored in the `occupiedCells` Map to prevent duplicate blocks at the same position.

## Voxel Grid

- 128×128 ground plane (default)
- Resized to `max(tts.w, tts.d)` when loading a prefab
- Blocks snap to a 1-unit grid
- Hidden `PlaneGeometry` serves as the ground-plane raycast target

## Three.js Axes

| Axis | Direction |
|------|-----------|
| X | East (+) / West (-) |
| Y | Up (+) / Down (-) |
| Z | South (+) / North (-) |

Standard three.js right-handed coordinate system.
