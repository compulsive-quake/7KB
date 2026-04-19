# Pregen World Format

How to ship a pregenerated world in `Data/Worlds/<Name>/` — what files the game needs, and the gotcha with `main.ttw`.

## Folder layout

Minimum files the game expects for a loadable world:

| File | Purpose |
|---|---|
| `map_info.xml` | World metadata: `HeightMapSize`, `Modes`, `Scale`, `RandomGeneratedWorld`, optional `FixedWaterLevel` |
| `dtm.raw` | Heightmap. `uint16 LE` per pixel; value = `Y * 256`. Size = `width * height * 2` bytes |
| `dtm_processed.raw` | Post-processed heightmap. For flat worlds, identical to `dtm.raw` |
| `biomes.png` | Biome color per pixel. MUST use a color registered in `biomes.xml` — see Biome color gotcha below |
| `radiation.png` | Radiation zones. Black = none |
| `splat1.png`, `splat2.png`, `splat3.png`, `splat4.png` | Raw terrain texture splat maps. RGBA, same dims as heightmap |
| `splat3_half.png`, `splat4_half.png` | Half-resolution splat maps. RGBA, half the heightmap dims |
| `splat3_processed.png`, `splat4_processed.png` | Runtime-ready splat maps. RGBA, same dims as heightmap |
| `water_info.xml` | `<WaterSources />` (empty is OK) |
| `prefabs.xml` | POI placements. `<prefabs />` (empty is OK) |
| `spawnpoints.xml` | Player spawn points. Needs at least one |
| `version.txt` | Freeform version/description string |
| `main.ttw` + `main.ttw.bak` | World header. **See gotcha below** |

Reference implementations: `Data/Worlds/Pregen08k01`, `Data/Worlds/Empty`, `Data/Worlds/Navezgane` under the game install dir. Smallest known-good world: `%APPDATA%/7DaysToDie/GeneratedWorlds/flatland` at 1024x1024.

### Minimum world size

`HeightMapSize` below ~1024 produces `Computed 0 chunk groups containing a total of 0 chunks` at `GenWorldFromRaw` time, with the same downstream symptoms as the biome-color gotcha (player falls forever, `TerrainData.SetHeights` NRE spam from `UnityDistantTerrain.CreateAndConfigureTerrain`). Observed at 128x128 with a valid `#ffa800` biomes.png, 238-byte main.ttw, and all eight splats present — the chunk grouper produced zero chunks anyway. Bumping to 1024x1024 fixes it. Exact minimum isn't characterized; 1024 matches the smallest shipped reference world and is a safe default for an empty arena.

### Biome color gotcha

`biomes.png` pixels must match a `biomemapcolor` declared in a loaded `biomes.xml`. Vanilla colors:

| Color | Biome |
|---|---|
| `#004000` | pine_forest |
| `#ba00ff` | burnt_forest |
| `#FFE477` | desert |
| `#FFFFFF` | snow |
| `#ffa800` | wasteland |
| `#001234` | underwater |

Using an unrecognized color (e.g. `#404040`) causes silent failure — no error is logged, but `GenWorldFromRaw` produces:

```
INF Computed 0 chunk groups containing a total of 0 chunks. Largest group contains  chunks.
```

The save directory is created, the player spawns, and then falls forever through an empty world, spamming `NullReferenceException: TerrainData.SetHeights` from `UnityDistantTerrain.CreateAndConfigureTerrain` and `[FELLTHROUGHWORLD] GetFallingSavePosition - closestChunk` (empty).

Mods *can* register new biome colors via `biomes.xml` XPath, but the pregen tooling needs that mod loaded at world-load time — it's easier to paint with a vanilla color and change surface appearance through the `dtm`/splat maps.

### Splat file gotcha

The game loads `_processed` variants at runtime but `ChunkProviderGenerateWorldFromRaw.processFiles` (the first-time world-load path) reads the raw `splat1`/`splat2`/`splat3`/`splat4` and `_half` variants too. If a world ships only `splat3_processed` + `splat4_processed`, new-game creation throws:

```
NullReferenceException: Object reference not set to an instance of an object
  at ChunkProviderGenerateWorldFromRaw+<processFiles>d__34.MoveNext
```

right after `INF Processing world files`. All eight splat PNGs must be present; transparent-black RGBA at the right dimensions is fine.

Note: this code path only runs when `main.ttw` parses successfully — a broken header silently skips `processFiles` entirely, masking the missing-splat problem until the header is fixed.

## `main.ttw` gotcha

This is a binary header parsed by `WorldState.SaveLoad`. It is NOT just magic + version — writing a truncated file causes:

```
ERR Exception reading world header at pos 25:
EXC Attempted to read past the end of the stream.
  at PooledBinaryReader.FillBuffer
  at WorldState.SaveLoad
WRN Failed loading world header file: .../main.ttw
INF Trying backup header: .../main.ttw.bak
ERR Exception reading world header at pos 25:
ERR Failed loading backup header file!
```

The game recovers and keeps running, but the downstream state is wrong — expect `[MultiBlockManager] Unexpected mode state. Current mode: Disabled. Expected mode: PrefabPlaytest.` spammed every frame, plus `ArgumentException: Attempting to create a zero length compute buffer` in `DynamicMeshManager`.

### Layout (empirical, from working files)

Version strings from older game versions are accepted — vanilla `Pregen08k01/main.ttw` ships as `V 2.0 (b282)` and loads fine under V 2.6. What matters is the **byte length**, not the version string.

```
offset  size  content
0x00    4     "ttw\0"                    magic
0x04    4     uint32 = 22                (0x16) format version
0x08    1     byte = N                    version string length
0x09    N     ASCII chars                 e.g. "V 2.5 (b32)"
...     4     uint32 = 1                  flag (observed constant)
...     4     uint32 = 2                  (observed constant)
...     4     uint32                      build-related
...     ...   more fields                 world dims, 0x427B851F magic, UUID...
```

Total size of known-good minimal files:
- `GeneratedWorlds/flatland/main.ttw` — 238 bytes
- `Data/Worlds/Empty/main.ttw` — 135 bytes (`Alpha 16(b15)`)
- `Data/Worlds/Pregen08k01/main.ttw` — 335558 bytes (carries region data)

### Practical approach

Don't hand-roll `main.ttw` from the spec — copy a known-good 238-byte template byte-for-byte. The game overwrites it when the first save is created, so the specific field values don't need to match your world.

PowerShell example:

```powershell
# 238-byte template from a working generated world
$mainTtwB64 = "dHR3ABYAAAALViAyLjUgKGIzMikBAAAAAgAAAAUAAAAgAAAAAAAAAAAAAAAAAAAAH4V7QhAAAAAQAAAAEAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAD//////////wAAAAA8AAAACgAAAAAAAAAAAAAAAAAAAAAAAADgLgAAAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAcAAAAHAAAABAAAAAAAAAAEAAAAAAAAAAQAAAAAAAAABAAAACAxQjY3NTNBOTRBRTY3RjRFOEFCRThFOUM4M0FFOEI2Nw=="
$bytes = [System.Convert]::FromBase64String($mainTtwB64)
[System.IO.File]::WriteAllBytes("$WorldDir\main.ttw", $bytes)
Copy-Item "$WorldDir\main.ttw" "$WorldDir\main.ttw.bak"
```

## Save vs world dir

`Data/Worlds/<Name>/` is the **template** read on new-game creation. The game writes a separate save at `%APPDATA%/7DaysToDie/Saves/<Name>/<GameName>/` which gets its own `main.ttw` (tagged with the current game version, e.g. `V 2.6 (b14)`). Broken template `main.ttw` only affects new-save creation; existing saves aren't touched.
