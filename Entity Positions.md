# Entity Positions

Part of the [7DTD Modding Knowledgebase](README.md). Covers how the game represents live entity positions at runtime, the origin-reposition trick used for floating-point precision, typical coordinate ranges, and common pitfalls when serializing positions to JSON.

For prefab grid coordinates (TTS voxel indexing, Y-offset, cell centering) see [Coordinate System](Coordinate%20System.md). For reading positions remotely over HTTP, see [7debug - Remote Debug Server](7debug%20-%20Remote%20Debug%20Server.md).

---

## `EntityAlive.position`

Every live entity in the world — players, zombies, animals, NPCs — is an `EntityAlive` and exposes a `Vector3 position` field maintained by the game's entity code. This is the entity's **world position** in the current game session, not a local transform:

```csharp
var world = GameManager.Instance?.World;
foreach (var kvp in world.Players.dict)
{
    var player = kvp.Value;
    var pos = player.position; // Vector3 — world coordinates
    var rot = player.rotation; // Vector3 — Euler angles
}
```

Accessing `position` on the main thread is safe. Accessing it from a background thread (e.g. an HTTP listener callback) is not — queue the read onto the Unity main thread if you need deterministic values.

### Axis conventions

| Axis | Direction | Typical range |
|------|-----------|---------------|
| `x` | East (+) / West (-) | `-3000..3000` depending on world size |
| `y` | Up (+) / Down (-) | Surface `30..80`, underground `< 0`, tall POIs `> 200` |
| `z` | North (+) / South (-) | `-3000..3000` depending on world size |

- A "grid reference" in 7DTD's HUD shows `x z` — the two horizontal axes.
- Sea level on default worlds is around `y = 32`.
- Spawn on Navezgane's starter city lands in roughly `(-112, 32, 1200)`.

### Rotation

`rotation` is an `Euler` vector. For "which way is the entity facing" use `rotation.y` (yaw) — this is what the HUD compass reads. `rotation.x` and `rotation.z` are pitch and roll respectively and are usually near zero for players on flat ground.

---

## Origin Reposition

Unity loses floating-point precision on transforms far from the world origin. 7DTD periodically shifts the **rendering origin** so the local player is near `(0, 0, 0)` in Unity scene space, then recomputes nearby transforms to keep physics and rendering accurate.

You will see lines like this in the game log whenever it happens:

```
8740+0 Origin Reposition (0.0, 0.0, 0.0) to (-112.0, 32.0, 1200.0)
```

This means the Unity scene root just moved. It does **not** mean the world moved.

### Why this matters (and why it probably doesn't)

**Good news:** `EntityAlive.position` is maintained by the game's entity code, not the Unity transform. It always reports real world coordinates. You do **not** need to add the reposition offset to values read via `player.position`.

**Where it bites you:**
- If you `Transform.position` on a GameObject directly, you're reading Unity local space and will get values near zero relative to the current origin.
- If you grep the log file for positions, the "Origin Reposition" line is describing a rendering-time shift, not the entity's location.
- If you do voxel math (block positions from world coords), the entity position → block index math is unaffected, but any Unity raycast or transform chain on top of it is.

The rule of thumb: read positions through `EntityAlive.position` (or a wrapper like [7debug](7debug%20-%20Remote%20Debug%20Server.md)'s `/api/players`) and treat origin reposition as a log artifact you can ignore.

---

## Sanity checks

When reading a position from the game, these checks will catch 90% of "why is the value weird" bugs:

| Observation | Likely cause |
|---|---|
| All three components are exactly `0` | Entity is in a "no world" state: main menu, loading screen, just disconnected. Treat the position as meaningless, not as "origin". |
| `y` is between `30` and `80` | Entity is on the surface. Normal. |
| `y < 0` | Entity is underground — a mine, a POI basement, or falling through the world. |
| `y > 200` | Entity is on top of a skyscraper POI, radio tower, or a mountain. |
| `x` or `z` beyond `±3000` | Entity is outside the configured world size — check `EnumGamePrefs.WorldGenSize`. |
| Values change by thousands in a single tick | Teleport, respawn, or a chunk load race. Don't interpolate across these. |

---

## Gotcha: C# interpolated strings with trailing `}}}`

Writing position JSON with an interpolated string looks obvious and breaks in a non-obvious way:

```csharp
// BROKEN — the z field prints "F1" instead of a number
sb.Append($"\"x\":{pos.x:F1},\"y\":{pos.y:F1},\"z\":{pos.z:F1}}}");
```

The parser treats `}}` inside the format specifier as an escape for a literal `}`, so the effective format string passed to `ToString` for `z` is `F1}`, which isn't a valid numeric format — it just echoes `F1` verbatim. `x` and `y` are fine because they're followed by `,`, not `}}}`. You end up with:

```json
{"x":-106.4,"y":37.7,"z":F1}
```

which isn't even valid JSON.

### Fix

Don't terminate an interpolation expression with a `}}}` sequence. Either build the field with separate `Append` calls:

```csharp
sb.Append("\"z\":");
sb.Append(pos.z.ToString("F1"));
sb.Append("}");
```

or use `string.Format` / `StringBuilder.AppendFormat` where the braces aren't ambiguous:

```csharp
sb.AppendFormat("\"z\":{0:F1}}}", pos.z);
```

This bug was observed and fixed in the 7debug player-list handler; if you're copying that handler as a starting point, double-check the z field of any position or rotation block.

---

## Distance and direction

### Horizontal distance

For most gameplay logic (pathfinding, aggro range, loot proximity), ignore `y` and use the 2D distance on the `x z` plane:

```csharp
float dx = a.x - b.x;
float dz = a.z - b.z;
float dist = Mathf.Sqrt(dx * dx + dz * dz);
```

### Compass bearing

Bearing from `a` to `b`, matching the in-game HUD compass where `0 = north`, `90 = east`:

```csharp
float bearing = Mathf.Atan2(b.x - a.x, b.z - a.z) * Mathf.Rad2Deg;
if (bearing < 0) bearing += 360f;
```

### Relative facing

To get "which way is the target relative to what the entity is looking at", subtract the bearing from the entity's yaw:

```csharp
float yaw = player.rotation.y;
float relative = (bearing - yaw + 540f) % 360f - 180f;
// -180 = directly behind, 0 = directly ahead, ±90 = hard left/right
```

---

## Related

- [Coordinate System](Coordinate%20System.md) — prefab voxel grid, cell centering, Y-offset
- [7debug - Remote Debug Server](7debug%20-%20Remote%20Debug%20Server.md) — reading positions (and the rest of game state) over HTTP
- [Entities](Entities.md) — defining custom entity classes via XML
- [Harmony Patching](Harmony%20Patching.md) — hooking entity methods (e.g. to observe damage, movement, or death)
