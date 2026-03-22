# X-Mirror Rotation

When prefabs are X-mirrored (X coordinates flipped), rotation indices must be corrected so blocks face the right direction in the mirrored coordinate space.

**Source**: `index.html` lines ~1112–1123

## X_MIRROR_ROT Lookup Table

```javascript
const X_MIRROR_ROT = [
//  input:  0  1  2  3    (Up)
    0, 3, 2, 1,
//  input:  4  5  6  7    (Down)
    4, 7, 6, 5,
//  input:  8  9  10 11   (North)
    8, 11, 10, 9,
//  input: 12 13 14 15    (South)
    12, 15, 14, 13,
//  input: 16 17 18 19    (East → West)
    20, 23, 22, 21,
//  input: 20 21 22 23    (West → East)
    16, 19, 18, 17,
];
```

Usage: `mirroredRotation = X_MIRROR_ROT[originalRotation]`

## Logic

Two transformations happen when mirroring across X:

### 1. Spin Reversal (within face groups)

Within each group of 4, indices 1 and 3 swap (90° ↔ 270°). Index 0 and 2 stay (0° and 180° are symmetric under X-mirror).

| Original | Mirrored | Why |
|----------|----------|-----|
| 0 (0°) | 0 (0°) | Symmetric |
| 1 (90°) | 3 (270°) | Mirror flips CW ↔ CCW |
| 2 (180°) | 2 (180°) | Symmetric |
| 3 (270°) | 1 (90°) | Mirror flips CW ↔ CCW |

### 2. East ↔ West Swap

The East group (16–19) and West group (20–23) swap entirely because mirroring across X flips the X-axis direction:

| Original | Mirrored |
|----------|----------|
| 16 (East 0°) | 20 (West 0°) then spin-reversed → 20 |
| 17 (East 90°) | 23 (West 270°) |
| 20 (West 0°) | 16 (East 0°) |
| 21 (West 90°) | 19 (East 270°) |

Up, Down, North, and South face groups stay in their own group (only spin is reversed).
