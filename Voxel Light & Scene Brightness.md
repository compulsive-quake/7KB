# Voxel Light & Scene Brightness

How to read "how bright is it here?" at runtime — and the gotcha that makes the obvious API wrong for it.

## The gotcha: `World.GetLightBrightness` is static exposure, not brightness

`World.GetLightBrightness(Vector3i)` → `Chunk.GetLightBrightness(x,y,z,0)` → `GetLightValue(...) / 15f`, where `GetLightValue` returns **`max(SUN channel, BLOCK channel)`** from the chunk's baked light data (0–15).

The SUN channel is Minecraft-style **sky exposure**, not sunlight *right now*:

- Open field at **midnight** → SUN = 15 → `GetLightBrightness` returns **1.0** ("bright").
- Under a bridge/roof at **noon** → SUN ≈ 0 (some horizontal propagation from the edges, −1 per block) → returns ~0 ("dark").

It is never modulated by time of day, so any "is it dark?" logic built on it fires under daytime overhangs and never fires outdoors at night. This exact bug caused the FPV mod's night-vision illuminator to white out the feed whenever the drone flew under anything in daylight. (FPV ultimately removed the night-vision boost entirely — a shadowless camera-mounted point light blows out nearby geometry no matter how it's gated — and instead relies on cloning the main camera's `PostProcessLayer` so the feed matches on-foot brightness; the recipe below is still the correct way to sample scene brightness if you need one.)

## Correct scene-brightness recipe

```csharp
// 0 at night .. ~1 midday (internally clamped to 0.7 then ×1.5 → 1.05 max;
// saturated through most of the day, ramps only near dawn/dusk)
float sun = Mathf.Clamp01(SkyManager.GetSunIntensity());

// Block light only (torches, lamps) — read the channel directly:
Vector3i blockPos = new Vector3i(worldPos);
IChunk chunk = world.GetChunkFromWorldPos(blockPos);
float blockLight = chunk != null
    ? chunk.GetLight(World.toBlockXZ(blockPos.x), World.toBlockY(blockPos.y),
        World.toBlockXZ(blockPos.z), Chunk.LIGHT_TYPE.BLOCK) / 15f
    : 0f;

float brightness = Mathf.Clamp01(Mathf.Max(blockLight, sun));
```

Notes:

- `IChunk.GetLight(int, int, int, Chunk.LIGHT_TYPE)` returns a byte 0–15; `LIGHT_TYPE` is `BLOCK` or `SUN` (nested enum on `Chunk`).
- `World.toBlockXZ(int)` / `World.toBlockY(int)` are the public static world→chunk-local coordinate helpers.
- Sample a block above your object if it hugs the ground/walls, or you may read a solid block (light 0).
- Voxel-only sampling **cannot** distinguish "under a sunlit overhang" from "inside a dark room" during the day — both have SUN ≈ 0. The sun-strength floor above trades away daytime-interior darkness detection to avoid false "night" under overhangs.

## Useful `SkyManager` statics (Assembly-CSharp)

- `GetSunIntensity()` — time-of-day sun strength, 0 .. ~1.05. **Not** weather-scaled (`weatherLightScale` only multiplies the actual `sunLight.intensity`).
- `GetSunPercent()` — raw sun elevation (`-sunDirV.y`), negative at night.
- `IsDark()` — dawn/dusk-time boolean the game uses for night checks.
- `GetMoonBrightness()`, `GetSunLightColor()`, `GetMoonLightColor()`, `GetSunAngle()`, `TimeOfDay()`, `GetDawnTime()` / `GetDuskTime()`.
