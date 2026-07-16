# Assetto Corsa Import (kn5, FSB5, engine audio)

How to turn Assetto Corsa car mods into 7DTD vehicles. Implemented end-to-end
by the **AssettoCar** utility (`C:\Users\Richard\WebstormProjects\7days\AssettoCar`
— Tauri app; Rust pipeline + Unity 2022.3 template project). Physics/config
donor is the [Supra](Vehicles.md) mod. AC *apps* (HUD widgets) can be ported
too — see [AC D Tachometer Port (IMGUI gauge HUD)](<AC D Tachometer Port (IMGUI gauge HUD).md>).

## AC car folder anatomy

```
content/cars/<car>/
  <car>.kn5          main model (LOD A); *_lod_b.kn5 etc. are distance LODs
  data.acd | data/   physics ini (encrypted acd; NOT needed if reusing donor physics)
  sfx/<car>.bank     FMOD Studio bank with all car audio + GUIDs.txt
  skins/<skin>/      livery texture overrides (match by texture file name)
  ui/ui_car.json     display name, brand, class (often malformed JSON — parse leniently)
```

## kn5 model format (versions 5/6)

`"sc6969"` magic, i32 version (v6 adds one extra i32), then:

- **Textures**: i32 count × { i32 active, str name, i32 size, raw bytes }.
  Strings are i32-length + UTF8 throughout. Blobs are usually DDS, sometimes
  PNG/JPG — sniff magic, don't trust the name's extension.
- **Materials**: i32 count × { str name, str shader, u8 blendMode (1 = alpha
  blend), u8 alphaTested, i32 depthMode, i32 propCount × { str name, f32 A,
  vec2 B, vec3 C, vec4 D }, i32 samplerCount × { str sampler, i32 slot,
  str textureName } }. Useful samplers: `txDiffuse`, `txNormal*`/`txBump*`,
  `txDetail`/`txNormalDetail`. Useful props: `ksSpecular`, `ksSpecularEXP`,
  `useDetail`, `detailUVMultiplier`, `detailNormalBlend`.
- **Node tree** (recursive): i32 type, str name, i32 childCount, u8 active.
  - type 1 dummy: 16×f32 local matrix, **D3D row-vector convention**
    (translation in floats 12..14). Loading the four float-quads as Unity
    Matrix4x4 *columns* makes `MultiplyPoint3x4` apply the same transform.
  - type 2 mesh: u8 castShadows, u8 visible, u8 transparent, u32 vertCount ×
    44-byte verts (pos 3f, normal 3f, uv 2f, tangent 3f), u32 idxCount × u16,
    u32 materialId, u32 layer, f32 lodIn, f32 lodOut, 16B bounding sphere,
    u8 renderable. **Skip meshes with `lodIn > 0`** (distance LODs).
  - type 3 skinned: rare in car bodies; parse to stay in sync, skip geometry.
- **UVs are DirectX-style** — flip `v' = 1 - v` for Unity. (DDS-in-Unity
  orientation vs this flip is UNVERIFIED against a real AC car as of
  2026-07 — first real import should check for upside-down liveries.)

### kn5 DDS blobs vs Unity's DDS importer (hit on first real car, 2026-07)

Unity's built-in DDS importer is far narrower than what AC cars actually
ship. Writing kn5 texture blobs straight to `Assets/**/*.dds` fails for
three whole classes (each failure logs 2 errors: `Unknown format` /
`Invalid mipCount …` + `Unsupported <path> file.`; the bundle still builds,
the textures are just missing):

- **RGBA-masked uncompressed 32-bit** (pixel-format masks R=0x000000FF,
  G=0x0000FF00, B=0x00FF0000, A=0xFF000000): rejected. Unity only accepts
  the classic BGRA/A8R8G8B8 masks (R=0x00FF0000). AC uses this for tire
  diffuse/normals (`pneuD/pneuN`), LCD, logos, `flat_nm`, damage/dirt masks.
- **A8L8 luminance** (16-bit, PF flags LUMINANCE|ALPHAPIXELS = 0x20001):
  rejected (`EXT_carbon_fiber`).
- **Partial mip chains on DXTn**: `Invalid mipCount: 5 provided, 10
  expected`. Unity demands a full chain down to 1×1 — or none: files with
  dwMipMapCount 0/1 import fine (it generates mips itself).

DXT1/DXT3/DXT5 with full or zero mips and BGRA-masked uncompressed all
import fine. Fixed in `AssettoCarBuilder.FixDdsForUnity` (unity-template):
swizzle R↔B + rewrite masks for the RGBA case, expand L8/A8L8 to BGRA32,
truncate partial chains to mip 0 (patch dwMipMapCount + caps, keep only
top-mip data — verified: all three repairs import cleanly, 28 errors → 0).
Anything else (odd masks, non-DXT FourCC like ATI2/DX10) is warned and
skipped so the error-counting headless gate stays green. Note DDS files
bypass `TextureImporter` entirely (IHV importer), so
AssettoTexturePreprocessor's normal-map flagging only ever applies to
PNG/JPG blobs — DDS normal maps import as plain textures.

### AC detail-map shaders: txDiffuse is often just an AO bake (hit 2026-07, bksy R34)

Binding only `txDiffuse` as albedo renders whole interiors (and most trim)
flat white, like a missing texture. On `ksPerPixelMultiMap*` /
`ksPerPixelMultiMap_NMDetail` materials with prop `useDetail=1`, the real
surface color is `txDiffuse × txDetail`, with txDetail tiled by
`detailUVMultiplier` (interiors 80–800×, carpaint ~80×). The R34 has 48/103
materials on this path: its dash diffuse is a white A8L8 AO bake
(`INT_cockpit_OCC.dds`) and the actual dark grey is a 4×4 `INT_grey.dds`
swatch tiled 300×. `txDetail = NULL.dds` (solid white) is a common no-op;
`detailUVMultiplier = 0` means "sample the corner texel" (solid-color
swatches), so keep tiling 0, don't default it to 1.

Fix in `AssettoCarBuilder.BuildMaterial`: bind txDetail as the Standard
shader's secondary albedo (`_DetailAlbedoMap` + keyword `_DETAIL_MULX2`,
tiling = detailUVMultiplier, `txNormalDetail` → `_DetailNormalMap` rides the
same UV transform). Standard multiplies by **2×detail**
(`unity_ColorSpaceDouble`), so set `_Color = 0.5` grey to land back on
diffuse×detail — exact in both gamma and linear space. Two exemptions:
- **The paint material** — the runtime owns its albedo and `_Color`
  (dye/livery swaps, and `Vehicle.SetColors` stomps `_Color` white at every
  spawn), so instead bake the detail's **average color** into a copy of the
  albedo (`BakePaintDetail`, skipped when the detail is near-white — e.g.
  the R34 ships a white base `metal_detail.dds` and ALL factory colors as
  skins → they arrive as color liveries instead).
- Reading pixels of imported textures: don't set `isReadable` (doubles
  runtime memory inside the bundle) — blit to a temporary RenderTexture and
  `ReadPixels` (works headless; the build already runs without
  `-nographics` for the icon render).

Also: the main normal-map binding must skip `txNormalDetail` (it starts
with `txNormal`, and dictionary order can make it win over the real
`txNormal` slot).

### Orientation without trusting the source convention

Standard AC node names `WHEEL_LF/WHEEL_RF/WHEEL_LR/WHEEL_RR` are dummy nodes
placed at the hubs. Derive vehicle space from them instead of assuming axes:
front-axle-midpoint − rear-axle-midpoint → snap yaw to 90° so the nose is
+Z; then if `WHEEL_LF.x > WHEEL_RF.x`, mirror X (negate x of positions and
normals, reverse triangle winding). Wheel radius = wheel-mesh bounds.y / 2.
Exclude node names containing `DAMAGE`, `BLUR`, `SHADOW`, `COLLIDER`, and the
low-res `COCKPIT_LR` / `STEER_LR` variants.

## FMOD .bank → WAV (FSB5: PCM16 + Vorbis)

AC banks embed one FSB5 blob. Two codecs seen in the wild: **codec 2 = raw
PCM16** (the Supra donor) and **codec 15 = Vorbis** — FMOD Studio's default
compression, common in modded cars (hit 2026-07, tatuusfa1_FormulaMT09; do
NOT assume PCM16). Find `"FSB5"`, header (`<4s I I I I I I`) =
magic, version, numSamples, sampleHeadersSize, nameTableSize, dataSize,
codec; sample data starts at `FSB5+60 + shSize + ntSize`. Per-sample u64:
bit0 = extra chunks follow, bits1-4 rate index (8=44100, 9=48000), bits5-6
channels (0=1,1=2), bits7-33 data offset in 32-byte units, bits34-63 sample
count. Chunk loop: u32 → bit0 more, bits1-24 size, bits25-31 type (1 =
channel override byte, 2 = rate override u32, 11 = VORBISDATA). PCM16 byte
length = min(next sample's offset, dataSize) − own offset, capped at
count×2×channels. Reference impls: `Supra/tools/extract_bank.py`,
`AssettoCar/src-tauri/src/pipeline/fsb5.rs`.

**FMOD Vorbis** is not an Ogg stream: the container and all three Vorbis
headers are stripped; the region holds raw audio packets, each with a u16le
length prefix (0 = end padding). The setup header (codebooks) exists only as
a CRC32 in chunk type 11 — decode requires the table of known libvorbis
setup headers keyed by that CRC (python-fsb5's `vorbis_headers.py`, MIT;
same trick as vgmstream/fsb-vorbis-extractor). Ident header is rebuilt with
blocksizes 256/2048; comment header is boilerplate. Rust impl:
`AssettoCar/src-tauri/src/pipeline/fsb5_vorbis.rs` (lewton raw-packet API +
bundled 620 KB table regenerated by `scripts/gen_vorbis_headers.py`; trim
decoded frames to the header's sample count). One CRC per (channels ×
quality) combo — a whole bank typically uses 1-2 headers.

### What sound AC actually plays — the FMOD event graph is the ground truth

A sample's ROLE is not its name — it's which FMOD **event** references it and
where on that event's parameter sheet it sits. `sfx/GUIDs.txt` lists the
events: the engine is exactly `event:/cars/<car>/engine_ext` (+ `engine_int`),
a single event whose **`rpms` parameter** crossfades between layered sound
modules. Every pop / backfire / limiter / shift / bov / turbo is a SEPARATE
event, so those samples are never on the engine's rpm axis no matter what
they're called. The compiled event graph lives in the bank's `BNKI`/RIFF
metadata (grep the bank for `rpms`), not the FSB5 audio — parsing it would
give the exact sample→rpm mapping + on/off-throttle layer, but the format is
proprietary and version-fragile (a real reverse-engineering project). We
approximate it from names + acoustics instead.

Bank sample naming (per soundset, wildly inconsistent between modders):
- **`ral`/`idle`** = idle loop (ralenti). One-shots keyed by substrings
  (`acc`, `bas`/`dec`, `gear`/`shift`, `valv`/`bov`, `horn`, `backfire`,
  `pop`, `growl`, `flutter`, `limiter`, `launch`, `turbo`, `coast` …).
- **Engine ladder, scheme A — numeric slice families** (Supra donor):
  `..._pot_Insert 5`, layers pot/pot2/adm/rev, trailing digit = slice number.
  Group by stripped-trailing-digits stem, mix same-numbered layers
  (primary 1.0, others 0.7, `rev` ramped in toward redline).
- **Engine ladder, scheme B — RPM band-word names** (McLaren 765LT
  `onvisu_mclaren_765lt_vesper`, hit 2026-07): `intverylow_012` /
  `newextlow_012` / `intlowmid_012` / `intmid_012` / `cmonhigh_012` — the
  `int`/`ext` prefix is mic position, `_012` is a shared take-id (NOT a slice
  number), and the rev band is a WORD. Order by
  `idle<verylow<low<lowmid<mid<midhigh<high<veryhigh` (separators stripped,
  most-specific word wins), one loop per rung (prefer the ext-mic take).

Classifier (`pipeline/sounds.rs` `classify`): expanded one-shot keyword
filter + an **acoustic duration gate** (`MIN_LADDER_SECS` 2.5s — a real
on-throttle loop runs seconds; a pop is <3s) decide the engine-loop
candidates, then scheme A is tried, then scheme B. This is what stops short
exhaust pops that happen to carry a trailing digit (`viejapop1/2/3`) from
being read as an rpm ladder — the exact bug the McLaren hit (pops mapped to
slices while the real V8 bands sat unused). f0 is still measured on the
chosen rungs to build the crossfade. Regression tests:
`src-tauri/tests/sound_classify.rs` (drives the public `describe_samples`;
NOT a lib unit test — see the comctl32 gotcha below).

### FMOD event-graph parse: ground-truth engine sample set (2026-07)

`pipeline/fmod_events.rs` recovers which FSB5 subsounds the engine actually
uses, from the compiled event metadata — no name guessing. The bank is a RIFF
`FEV ` file; the relevant records (walk the LIST tree, chunks pad to even):

- `WAV ` (in `WAVS`): 16-byte waveform GUID + FSB5 subsound index at
  **body+22**. One per subsound; subsound index == name-table index == the
  order `extract_samples` returns, so it doubles as a sample index.
- `WAIS/WAIT`: each engine layer is a `WAIB` then an `INST`. The WAIB's
  **second** GUID (body+16) resolves through the WAV table to a subsound;
  the WAIB's first GUID does not. Thread the "current WAIB waveform" through
  the walk (reset at each `WAIT`) — a layer's INST can sit one list-level
  below its WAIB.
- `PRMB` (in `PRMS/PARM`): parameter records. The engine parameter is named
  `rpms` (length-prefixed, followed by its float range ≈14976–20000). An
  `INST` whose body embeds an rpms-parameter GUID is rpm-driven = an engine
  layer; everything else (backfire/horn/surfaces on their own events) is not.

Gotchas that cost time:
- **A bank has MORE than one `rpms` parameter** — one per engine event
  (engine_ext + engine_int). Collect them ALL and match an INST against any;
  taking only the first drops a whole event's layers (the Supra's `rev`
  family, referenced by the second rpms param, vanished → 66 samples instead
  of 83). See `GUIDs.txt` for the event list (`event:/cars/<car>/engine_ext`
  etc.) — engine is always exactly `engine_ext`/`engine_int`.
- Chunk walk must bound only by the FILE length, not the parent chunk's
  declared size; FMOD container sizes can be tighter than their content and
  a parent-overflow break aborts the sibling scan.
- Verified: Supra bank → 83 engine samples (pot/pot2/adm/rev + idle + accel
  one-shots), McLaren → 20 (int+ext rev-band layers + coast/shift + idle),
  and pure one-shots (backfire, horn, doors) correctly excluded. Test:
  `src-tauri/tests/fmod_events.rs`.

### Per-sample rpm axis (2026-07, `engine_layers`)

The centre rpm of each engine layer is the **f32 exactly 16 bytes after the
rpms-parameter GUID inside that layer's `INST` body** — clean authored values
(idle ~1000, redline layers ~6500). `fmod_events::engine_layers` returns
`(sample, rpm)` per layer (median rpm if a sample is placed in both
engine_ext and engine_int). Verified: McLaren idle 1190 / verylow 1700 /
low 2000 / lowmid 3200 / mid 4000 / high 5500–6600; Supra `ral` 1000, slices
1280 → 9150. Crucially **same-rpm samples are the layers of one rung** — a
Supra slice's pot/pot2/adm/rev all share its rpm (slice 4 = 1280, slice 5 =
1540), so grouping by rpm reconstructs the layer structure for free.

**How it's wired (`sounds.rs` `detect_ladder_fmod`, ground-truth-first):** when
`engine_layers` is available the ladder is built straight from the rpm axis —
engine loops keyed by true centre rpm (BTreeMap → true order), same-rpm takes
grouped as one rung (primary = exhaust/pot else ext-mic else longest; `rev` =
ramp-in layer). `is_loop` still drops the coast/shift/idle samples the rpms
event also drives (7DTD's VPEngine crossfade is one loop per rpm). Only if
that yields <3 rungs does it fall back to the name/acoustic heuristic, so it
never regresses. McLaren went 5 heuristic rungs → 7 ground-truth rungs
(1700–6600). Review UI logs "Engine ladder from the FMOD event graph: N
rungs, LO–HI rpm (ground truth)".

f0 is STILL what drives the crossfade: the VPEngine `SoundRange` math is about
audio pitch (playback-rate ratios of each loop's fundamental), so f0 is the
right quantity there; rpm gives bulletproof ORDER + layer grouping + idle, and
f0 is measured per rung as before. The full fade min/max (not just centre)
lives in the `CTRL`/`CURV` automation curves (18, keyed to the rpms GUID) and
is still undecoded — not needed, since f0 sets the fades. Diagnostic scripts:
`scripts/fmod_engine_ladder.py` (ground-truth ladder + rpm), `probe_rpm_range.py`,
`explore_fmod_bank.py`.

Still unparsed naming (only matters when the FMOD parse fails, e.g. a
non-Studio bank): the tatuusfa1 `v6_low_2halfk`/`5k`/`9halfK` scheme
(rpm-as-`Nk`-suffix) → donor-engine fallback.

## VPEngine gear/SoundRange format — CRACKED (V2.x decompile)

`vehicles.xml` engine `gearN` value, comma-separated:

```
rpmMin,rpmMax, rpmDecel,rpmDownShiftPoint,rpmDownShiftTo,
rpmAccel,rpmUpShiftPoint,rpmUpShiftTo, accelSound, decelSound,
then per loop range (8 fields):
pitchMin,pitchMax, volumeMin,volumeMax, pitchFadeMin,pitchFadeMax,pitchFadeRange, name
```

Runtime (`VPEngine.updateEngineSounds`): `pitch = lerp(pitchMin, pitchMax,
rpmPercent)` is the Handle pitch (offset from base 1.0); volume =
`lerp(volMin, volMax, rpmPercent)`, then faded to 0 over `pitchFadeRange`
once pitch leaves `[pitchFadeMin, pitchFadeMax]`. Loops below 0.01 volume
are stopped, so inaudible ranges cost nothing.

**Generation recipe for an rpm-slice ladder** (verified against every number
in the Supra's shipped XML): with band `fLow` (idle f0) … `fHigh` (top
anchor f0), for an anchor with fundamental `f0`, neighbors `fPrev`/`fNext`:

- `pitchMin = fLow/f0 − 1`, `pitchMax = fHigh/f0 − 1` (anchor plays at
  native pitch exactly at its own rpm)
- `pitchFadeMin = (fPrev/f0)^¼ − 1` (idle: −9 = never fade at bottom)
- `pitchFadeMax = (fNext/f0)^¼ − 1` (top anchor: 9)
- `pitchFadeRange = (fPrev/f0)^¼ − (fPrev/f0)` — fade starts at the
  quarter-log-point toward the neighbor and dies exactly at the neighbor's
  fundamental. Idle uses the next-anchor equivalent:
  `range = (f1/fLow − 1) − ((f1/fLow)^¼ − 1)`.
- Volumes: idle `1,1`, anchors `.8,.95`.

Measure f0 by normalized autocorrelation (mid-file 1 s window, 25–600 Hz
lags, parabolic refine, prefer the octave-up while it correlates ≥ ~0.92 of
the best lag — engine loops love subharmonic locks).

### Runtime gear sim is a TIME ramp, not speed (V2.x decompile)

`VPEngine.updateEngineSimulation` (fired per frame from
`Vehicle.UpdateSimulation`, both local-drive and remote paths) revs a
synthetic rpm by `rpmAccel * dt` while `CurrentIsAccel`, upshifts on
`rpmUpShiftPoint` (gated on `CurrentForwardVelocity > 4`), and decays by
`rpmDecel * dt` to gear 0 / idle the moment throttle lifts — even at top
speed. It is completely uncorrelated with road speed, so it will never match
a speed-derived HUD tach. `rpmPercent = (rpm − rpmMin)/(rpmMax − rpmMin)`
of the *current XML gear* feeds `updateEngineSounds`, so every XML gear
sweeps the full slice ladder.

**Sync pattern** (Supra `SupraEngineSoundPatch` + `SupraDrivetrain`): Harmony
prefix `updateEngineSimulation`, return false for your vehicle, compute your
own speed-based smoothed rpm, map it into the ladder band
(`percent = (rpm − fLow·20)/(fHigh·20 − fLow·20)` for an I6 where
display rpm = f0·60/3), call `__instance.updateEngineSounds(percent)`
directly (publicized). Pin `__instance.gearIndex = 0` when all XML gears
share identical ranges so loop Handles live in one place. Fire
`playAccelDecelSound(...)` yourself on your model's shift events (it stops
the previous one-shot — rapid restarts read as quick shifts). The turbo
+0.2 pitchAdd and `vehicle_turbo` one-shot live inside `updateEngineSounds`
and are kept for free. A full decompile lives at
`Elevator/.claude/tmp/decomp/` (VPEngine.cs, Vehicle.cs, EntityVehicle.cs).

**Generalized into AssettoCar (2026-07):** ported to `AssettoDrivetrain.cs` +
`AssettoEngineSoundPatch.cs` in the shared runtime. Two things the Supra
hardcoded had to go car-agnostic because a generated car's slice ladder
(count + fundamentals) varies: (1) the rpmPercent map is just
`(rpm − idle)/(redline − idle)` clamped 0..1 — rpmPercent is only a normalized
position in whatever ladder the XML gears define (idle loop at 0, top anchor
at 1), so idle→redline sweeps the full ladder for ANY car; drop the I6
`f0·20` band. (2) The one-shot clip names are per-car — the DLL is shared, so
reconstruct the prefix from the entity class name: generated mods name the
class `vehicle<Id>` and the sound nodes `<id>_accelN`/`<id>_decelN` with
`id = Id.ToLowerInvariant()` (Rust `Naming::derive`), so
`className.Substring("vehicle".Length).ToLowerInvariant()` rebuilds it. The
drivetrain constants (redline 7000, idle 850, shift 3600, 8-speed band table
to 69.3 m/s) carry over unchanged because every generated car reuses the donor
physics tune. Vehicle identity is `AssettoPaint.IsOurs(ev)` (type-name check),
not a hardcoded class name.

### updateEngineSounds, verified line-for-line (2026-07)

Re-read against the decompile while building the drive preview; the
generation recipe above is confirmed, plus three details worth knowing:

- **Pitch is an offset, not a rate.** `Audio/Handle.SetPitch` does
  `source.pitch = pitch + basePitch`, and nothing in a generated
  `sounds.xml` sets a base, so **playback rate = 1 + pitch + pitchAdd**.
  That's why the generation formula subtracts 1 everywhere.
- **`Mathf.Lerp` clamps.** Both the pitch/volume lerps and the fade
  (`Mathf.Lerp(1, 0, over / pitchFadeRange)`) clamp t, so rpmPercent outside
  0..1 and fades past the neighbour just saturate. Don't add your own
  `abs()`/epsilon to `pitchFadeRange` — a negative range is a real "never
  fade" case the clamp already handles.
- **There is a constant slow detune.** `pitchRand` re-rolls to
  ±0.03 every 0.75 s and `pitchAdd = MoveTowards(pitchAdd, pitchRand, dt*0.15)`
  crawls toward it; turbo adds +0.2 to the same value (and fires the
  `vehicle_turbo` one-shot). Loops under **0.01** volume are stopped.

### Drive preview (`pipeline/drive_preview.rs`, 2026-07)

The sounds page renders "hold W on flat asphalt" offline and plays it under a
synced tachometer. It is a real re-implementation, not an impression: clips
come from `sounds::build_soundset`, the crossfade numbers from
`modgen::sound_ranges` (one shared `SoundRange` model now feeds both the XML
writer and the renderer — that's the point), evaluated by `SoundRange::eval`
(the port of `updateEngineSounds`), driven by `AssettoDrivetrain`'s
speed→gear→rpm model. Only road speed is modelled — see the Vehicles note's
"How speed actually happens".

Sanity numbers for the donor tune, hold-W, no turbo: upshifts at 9/14/20/27
m/s (the `GearTop` band edges), settles at the 33 m/s cap ≈ 5870 rpm in 5th,
~12 s of audio. Turbo settles ~64 m/s (drag beats the 69.3 cap). Tests in
`tests/drive_preview.rs` pin the eval math (anchor plays native at its own
rpm; each anchor silent by its neighbour's fundamental; no silent hole
anywhere in the band) — those are the tests to run after touching the ladder.

## Assembly into a 7DTD mod

Everything else is the [Vehicles](Vehicles.md) + [Asset Bundles](Asset%20Bundles.md)
playbook: SetupSupraPrefab's prefab contract (Physics/Wheel0..3/`GameObject`/M),
Supra donor vehicles.xml (1550 kg tune — keep the prefab mass in sync),
sounds.xml SoundDataNodes (flat unique names!), per-mod bundle+prefix names
so multiple imported cars coexist. The color picker is generalized into
`AssettoCarRuntime.dll` (shared by every generated mod; AppDomain marker
guards double-load, paint json in the save folder is mtime-reloaded so
multiple DLL copies stay coherent).

**Supra feature parity ported into the generator (2026-07)** — every newly
generated mod now ships these (existing mods must be regenerated to get them):
- **Full-time AWD default**: `vehicles.xml` wheel0/wheel1
  `torqueScale_motor_brake = 1,1` (was `.5,1` rear-biased) so the launch
  doesn't wheelspin on RWD alone.
- **Side grip**: `AssettoCarBuilder.cs` WheelCollider `sidewaysFriction.stiffness`
  3.0 (was 2.0). Prefab-baked, so it applies per generated bundle (no separate
  rebuild — bundles build per-car at generation time).
- **RPM-synced engine audio + Initial-D tachometer HUD**: `AssettoDrivetrain`,
  `AssettoEngineSoundPatch`, `AssettoTachometer`, `AssettoHudClearance`. Themes
  (D3–D7) ship at `<mod>/Tachometer/themes/` — modgen copies the whole
  `resources/mod-template/Tachometer/` tree per mod; the runtime loads them from
  `AssettoCarModApi.ModPath`. Tach on/off + unit + theme picker live in the
  paint dialog; settings persist per save in `assettocar_tachometer.json`
  (shared across all generated mods, like paint). See
  [AC D Tachometer Port](<AC D Tachometer Port (IMGUI gauge HUD).md>).
- **Chunk pre-load ahead**: `AssettoChunkPreloader` (ticked from `AssettoHost`)
  adds a second [chunk observer](<Chunk Observers & Chunk Loading.md>) led along
  the velocity vector (2 s, capped 96 m) so road decorations don't pop in at
  speed; SP/integrated-host only (`ConnectionManager.IsServer`).

### Livery system (added 2026-07)

AC skins are texture overrides by exact file name in `skins/<skin>/`; only
the paint material's albedo matters for body liveries. Pipeline: the Unity
build report carries `paintTexture` (the kn5 texture name the paint material
uses — mirrors BuildMaterial's txDiffuse-else-first-slot pick), then
`pipeline/liveries.rs` matches each skin folder against it (exact name, then
same stem with dds/png/jpg), decodes (image_dds handles BC1-7 + uncompressed
BGRA8), downsizes to ≤2048 and ships JPEG q90 into `<mod>/Liveries/` plus the
skin's tiny `livery.png` as picker icon, described by `Liveries/liveries.json`
(`{entityClass, liveries:[{id,name,file,icon}]}`; name from ui_skin.json
`skinname` else prettified folder name). Livery extraction failures degrade
to warnings — the mod still works without them.

**Color skins with no albedo override (added 2026-07, hit on bksy R34
V-Spec II Nür):** many cars' skins never touch the paint albedo. Kunos-style
carpaint shaders (`ksPerPixelMultiMap` etc., `useDetail=1`) take the body
COLOR from the heavily tiled **`txDetail`** texture
(`detailUVMultiplier`≈80, e.g. `metal_detail.dds`) — skins recolor the car
by overriding THAT file, so the Unity report now exports
`paintDetailTexture` alongside `paintTexture`, and the extractor turns a
detail-override skin into a **color livery** whose value is the texture's
average RGB (detail maps are near-flat color fields; avg = the paint).
`cm_skin.json` (`carPaint.color` as **`#AARRGGBB`**, alpha first — strip
it; `enabled:false` = don't repaint) is CM paint-shop *metadata* and is
frequently stale (the R34's Lightning Yellow says white!) — use it only as
a last-resort fallback when no texture matches; `ext_config.ini` has only
specular/flake params, no diffuse color. Priority: albedo override (texture
livery) > detail override (avg-color livery) > cm_skin.json color. The
runtime applies color liveries like the color picker (albedo→white,
`_Color`=the paint) while keeping the livery id selected in the picker;
liveries.json entries carry `file` XOR `color`. A skin with none of the
three is skipped.

Runtime: `AssettoLiveries.cs` scans ALL loaded mods' `Liveries/liveries.json`
(so the one hosting DLL copy serves every generated mod), maps vehicles via
`EntityClass.list[ev.entityClass].entityClassName`, lazy-loads textures with
`Texture2D.LoadImage` (needs `UnityEngine.ImageConversionModule` reference)
and DXT-compresses when dimensions are multiples of 4. `AssettoPaint` stores
per-entity `livery:<id>` OR `#RRGGBB` in the same save-folder json (exclusive
states); livery apply = albedo→livery tex + `_Color`=white, and the factory
albedo snapshot must never cache white/one-of-our-livery textures (guard via
`AssettoLiveries.IsLiveryTexture`) or "Factory paint" restores the wrong skin
after a save/load. The paint dialog shows a livery column when the manifest
exists; while a livery is active the HSV wheel/hex controls are disabled
("Custom color" list row re-enables them).

Gotchas hit while building this:
- Tauri v2 *test* executables die at load with STATUS_ENTRYPOINT_NOT_FOUND
  (0xc0000139): tao/muda import comctl32 **v6-only** named exports
  (`SetWindowSubclass`, `TaskDialogIndirect`, `DefSubclassProc`), and a bare
  test exe has no manifest declaring the Common-Controls 6.0.0.0 dependency,
  so the loader binds comctl32 5.82. The main app is fine (tauri-build embeds
  a manifest). An external `.exe.manifest` does NOT work — rustc embeds a
  default manifest since 1.71, which wins. Fix in build.rs:
  `cargo:rustc-link-arg-tests=/MANIFEST:EMBED` +
  `cargo:rustc-link-arg-tests=/MANIFESTINPUT:<path to manifest with the
  Common-Controls dependency>` (MSVC merges all MANIFESTINPUTs). See
  `AssettoCar/src-tauri/tests/common-controls.manifest`. **`-tests` only
  covers `tests/` integration crates, NOT the `--lib`/`--bins` unit-test
  harness** — a lib unit test loads fine until enough pipeline code is pulled
  in to make the linker retain tao's window code (and its comctl32-v6
  imports), then that exe fails 0xc0000139 while `tests/` stays green. So put
  pipeline tests that touch much of the crate in `tests/` and drive the
  public API (e.g. `sound_classify.rs` → `describe_samples`), not in a
  `#[cfg(test)] mod` inside the lib.
- No AC install exists on this machine (checked 2026-07: zero .kn5 anywhere);
  `AssettoCar/tools/make_fixture.py` writes a synthetic kn5 car for testing
  and hardlinks the real Supra AC bank for the audio path. Real AC cars live
  in `C:\Users\Richard\Desktop\cars\`, generated mods in
  `C:\Users\Richard\Desktop\mods\` (copied manually into the game's Mods
  folder — AssettoCar output is not modman-managed).
- Debug helpers: `AssettoCar/tools/dump_kn5_mats.py <car.kn5> [name-filter]`
  prints a kn5's full material/texture tables (shader, props, sampler
  bindings, blob formats). `src-tauri/tests/convert_real.rs` converts a real
  car end-to-end without the UI: set `AC_CAR_PATH` (+ optional
  `AC_MOD_NAME`/`AC_OUT_DIR`), then
  `cargo test --test convert_real -- --ignored --nocapture`.
