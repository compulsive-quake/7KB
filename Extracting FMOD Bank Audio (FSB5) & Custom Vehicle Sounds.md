# Extracting FMOD Bank Audio (FSB5) & Custom Vehicle Sounds

How to pull WAVs out of an FMOD Studio `.bank` (e.g. an Assetto Corsa car
soundset) and wire them up as a 7DTD vehicle's engine/horn sounds. Verified
July 2026 building the Supra mod's 2JZ soundset from
`paris_toyota_supra_a90_s1.bank`.

## .bank anatomy

An FMOD Studio bank is a `RIFF`/`FEV` container (event/mixer metadata) with a
single embedded **FSB5** sample blob — find it with `data.find(b'FSB5')`.
FSB5 header (60 bytes, version 1): `4s magic, u32 version, numSamples,
sampleHeadersSize, nameTableSize, dataSize, codec`. **Codec 2 = raw PCM16** —
Assetto Corsa banks use it, so extraction is pure Python (no Vorbis). Codec 15
= Vorbis → use vgmstream-cli instead (see the Wwise note for the download).

Per-sample header is a u64 (layout per vgmstream `fsb5.c` — several Python
libs get this wrong): bit 0 = has-extra-chunks, bits 1–4 = rate index
(8=44100, 9=48000), bits 5–6 = channel mode (0=1ch, 1=2ch), bits 7–33 =
data offset in **32-byte units**, bits 34–63 = sample count. Extra chunks
follow while bit 0 of a u32 is set: size = `(c>>1)&0x7FFFFF`, type =
`(c>>25)&0x7F` (1=channels override, 2=freq override, others skippable).
Name table: `numSamples` u32 offsets, then NUL-terminated strings. Sample
byte-length = next sample's offset − own offset (offsets are sorted &
32-byte aligned), capped to `count*2*channels`.

Working extractor: `Supra/tools/extract_bank.py` (bank → named WAVs).

## AC soundset naming (French!)

- `ral` = **ralenti = idle loop** — the key find.
- `pot`/`pot2` = exhaust RPM loops, `adm` = intake loops, `rev` = top-end
  layer, each as `Insert 1..18` RPM slices (~6.2 s seamless loops, low→high).
- `*acc01..08` = on-throttle rev loops low→high; `*bas*` = off-throttle.
- `pet*` = pétarade (backfire pops), `valv*` = blow-off (decaying — good
  shutoff), `rupt` = limiter, plus `horn`, `gear_up/dn`, `skid`, `wind`, etc.
- Audio-blind sanity checks that worked: RMS envelope in fifths (steady =
  loop, decaying = shutoff-shaped) and zero-crossing rate to rank RPM slices.
- **Third naming convention: rpm-numbered takes** (bksy Mazda ND, 2026-07-18):
  samples named by their centre rpm plus a take letter — `3068a`, `5531a`,
  `idle_1332`, mic variants like `6419b_inside`, and **off-throttle takes
  suffixed `_off`** (`3917b_off`). The `_off` takes are overrun recordings
  and must be excluded from the on-throttle crossfade ladder (an even-spread
  rung pick once chose `5531a_off` OVER `5531a` — the engine went limp
  mid-pull). The idle's true rpm comes from the FMOD event graph (1318 for
  the ND) or the name digits; AC cars do not all idle at ~850.
- **Names lie — always duration-gate transient roles** (2026-07-17): the ASW
  McLaren 765LT's `growlshiftpop02` is a 45 s on-throttle growl LOOP, not a
  shift pop. Name-matched as the "gear click" and layered onto the shift
  one-shots it played as a constant unreal whine over the whole pull. A real
  click is ≤ ~2.5 s, blow-off ≤ ~3 s, accel/decel one-shots ≤ ~8 s
  (AssettoCar `sounds.rs::classify` now enforces these). Related: when
  mixing multi-layer rungs, trim to the SHORTEST layer — padding to the
  longest leaves a tail where one quiet layer plays alone, which becomes a
  periodic dropout in a loop; and an overlay must never extend its base clip.

## Wiring into a 7DTD vehicle (XML only)

1. Ship clips as WAVs **inside the mod's asset bundle** (an
   `AssetBundleBuild.assetNames` list can mix a prefab + AudioClips; asset
   name for lookup = file name without extension).
2. `Config/sounds.xml`: `<append xpath="/Sounds">` with `SoundDataNode`s
   cloned from the vanilla SUV nodes (`Data/Config/sounds.xml` ~line 13100),
   `ClipName="#@modfolder:Resources/<bundle>.unity3d?<clip>"` — clips load
   via `DataLoader.LoadAsset<AudioClip>`, so the modfolder syntax works.
   Reuse vanilla `AudioSource name="@:Sounds/Prefabs/AudioSource_Vehicle.prefab"`
   (`AudioSource_SmallHorn.prefab` for horns). `Loop="true"` on loop clips.
   Keep the `<Noise>` lines or the vehicle becomes silent to zombies.
3. `vehicles.xml` engine part: `gear1..4` strings embed
   `accelSound, decelSound, ` then **any number** of 8-float sound-range
   groups (`(tokens-10)/8`), each ending in a loop name; plus `sound_start`,
   `sound_shut_off`, and vehicle-scope `hornSound`.

### Sound-range semantics (decompiled `VPEngine.updateEngineSounds`)

Per range: `pitchMin,pitchMax, volMin,volMax, fadeMin,fadeMax,fadeRange, name`.
`rpmPercent = (rpm - gear.rpmMin)/(gear.rpmMax - gear.rpmMin)` (0 at idle);
pitch and volume lerp by it. **Pitch values are OFFSETS from the AudioSource
prefab's base pitch 1.0** (`Handle.SetPitch` does `pitch + basePitch`) — that's
why vanilla max ranges are negative (`-.4` = 0.6× playback). The fade pair is
a window in pitch-offset units: the loop fades to 0 over `fadeRange` when its
current pitch falls below `fadeMin` or rises above `fadeMax`; volume < 0.01
stops the handle. Design consequences (Supra 2026-07):

- Vanilla's 2-range gears leave a **mid-rpm hole**: idle (pitch window
  `.12+.1`) is gone by rpm ≈ .3 while the max loop (fade-in below `-.2`)
  isn't full until rpm ≈ .5 — and each upshift resets the virtual rpm to
  ~15% of the band. Reads as "engine goes away while driving".
- A single AC layer as a loop sounds thin — **mix the loop-aligned RPM-slice
  layers offline** (`pot + pot2 + adm (+ rev near redline)`; same slice
  number = same length, loops stay seamless) into each shipping loop.
- **The endgame fix: emulate FMOD's RPM-slice crossfade directly in the
  range system** (this is what finally sounded right). AC records the
  engine at ~18 fixed RPMs and crossfades neighboring slices, bending each
  only a few % in pitch — never stretching one loop across the band.
  7DTD's ranges express this exactly:
  1. Measure each slice's fundamental f0 (numpy FFT + harmonic product
     spectrum; repair octave-glitched cells against the other layer
     families — e.g. intake often reads an octave low). The slices form a
     ladder (Supra bank: 37→457 Hz over 18 slices; `ral` idle = 57.2 Hz).
  2. Ship K anchors (idle + every other slice), map the gear band linearly
     onto `f_lo..f_hi` Hz, then per anchor: `pitchMin = f_lo/f0 − 1`,
     `pitchMax = f_hi/f0 − 1` (pitch offsets are linear in rpm, so each
     anchor crosses native 1.0× exactly at its recorded rpm),
     `fadeMax = (√r_next − 1)/2`, `fadeMin = (1/√r_prev − 1)/2`
     (handoff at the log-midpoint between anchors), `fadeRange =` the
     smaller of the distances to the neighbors' native pitches (so each
     loop is silent by the time its neighbor is native).
  3. Simulate before shipping: replay `updateEngineSounds` math over
     p = 0..1 and check the summed volume has no holes (target ≈ 0.85–1.25)
     and per-loop playback stays within ~0.85–1.2×.
  Inactive ranges cost nothing (vol < .01 stops the handle); ~2 loops play
  at any instant. Same 8-range stack in every gear — each gear sweeps the
  full ladder, which fits the arcade rev model.
- Blow-off valve on throttle lift: decel one-shots fire when rpm decays to
  `rpmDownShiftPoint`, so overlay the BOV whoosh on the decel clips' heads —
  NOT on `sound_shut_off` (that plays when the engine turns off / on exit).
  With no engine-off recording in the bank, render shutoff as an idle
  spin-down (resample with falling rate + fade — see make_vehicle_sounds.py).

**Name resolution gotcha:** `Audio.Manager.ConvertName` ends with
`StripOffDirectories` (`Path.GetFileName(name).ToLower()`) on **both**
registration and lookup — vanilla's `Vehicles/Suv/suv_accel1` references
resolve to the node named `suv_accel1`. So directory prefixes in
vehicles.xml are cosmetic, and custom node names must be **flat and
globally unique** (`supra_idle_lp`, not `Vehicles/Supra/idle_lp`).

Other facts learned decompiling `VPEngine` (the engine part class —
there is no `VehiclePartEngine`):

- Accel/decel are one-shots via `entity.PlayOneShot`; loops via
  `Manager.Play(entity, name, wantHandle: true)` with pitch/volume driven
  per-frame from the sound-range floats.
- The turbo sound is a **hardcoded global** node name `vehicle_turbo` —
  overriding it affects every vehicle; there's no per-vehicle slot.
- 7DTD has no gearshift sound slot; accel one-shots fire on gear change, so
  bake a shift click into the head of accel2/3 WAVs if you want audible
  shifts (`Supra/tools/make_vehicle_sounds.py` overlays `gear_up` at 70%).
- Loudness: match clips to a common RMS (~-16 dBFS, peak-limited) and let
  the XML range volumes do the balance — AC raw samples vary wildly.

## Related

- [Extracting WEM Audio (Wwise)](Extracting%20WEM%20Audio%20(Wwise).md) — same job for Wwise containers (vgmstream)
- [Vehicles](Vehicles.md) — the rest of the vehicles.xml schema
- [Asset Bundles](Asset%20Bundles.md) — building/loading `#@modfolder` bundles
