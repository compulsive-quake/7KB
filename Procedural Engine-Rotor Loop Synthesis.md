# Procedural Engine/Rotor Loop Synthesis

A code-only 7DTD mod can synthesize a looping motor/rotor sound entirely in
C# — no `.wav`, no asset bundle — and drive its pitch/volume live. This is how
the FPV mod's drone motor works (`FPV/src/FPVDroneAudio.cs`), not a sampled
clip. Same runtime `AudioClip` path as [[Runtime WAV Loading (AudioClip from disk)]],
just filled by a synth instead of a file.

## Recipe (see `BuildRacingLoopClip` / `BuildTinyLoopClip`)
1. Pick a loop length `L` seconds; `count = ceil(sampleRate * L)`.
2. **Seamless-by-construction tonal bed:** snap every tonal frequency to an
   integer number of cycles over the loop, i.e. to the grid `1/L` Hz
   (`round(freq / grid) * grid`). A grid-snapped sine wraps with zero
   discontinuity, so no crossfade is needed on the tonal layer.
3. Per motor (4 for a quad), sum a harmonic series `Σ harm[h]·sin(2π·f0·(h+1)·t)`.
   Detune the four motors a few percent so their tones beat against each other
   (the "angry swarm" shimmer). Add a once-per-rev wobble + low hum via the
   rotation-rate sine.
4. **Prop wash:** differentiated white noise (crude high-pass) → two-pole
   lowpass → amplitude-chop at the blade-pass rate. The noise layer is NOT
   grid-aligned, so crossfade its head into its tail (~1024 samples) to loop it.
5. Mix `SoftClip(tonal·a + wash·b)` (tanh soft clip), `Normalize` to ~0.9 peak,
   `AudioClip.Create(name, count, 1, sampleRate, false)` + `SetData`.
6. **Live pitch = rpm model.** Render the clip at a reference RPM
   (`RenderRpm`), then at runtime set `source.pitch = rpmNow / RenderRpm` where
   `rpmNow` chases throttle (smoothed). Unity clamps `pitch` to ±3, so keep the
   full-send multiplier under 3. Volume tracks `sqrt(rpm)`.

## Timbre knobs (what makes it read big vs small)
- **Fundamental / base freq:** higher = smaller airframe. A 5" quad's blade
  pass ≈ `RenderRpm/30`; the tiny whoop uses ≈ `RenderRpm/11.5` (~2.6x higher).
- **Harmonic rolloff:** fundamental-dominant + fast rolloff = fat *buzz*;
  near-flat rolloff with hot upper partials = thin, *screechy* whine. Hot upper
  harmonics + tanh drive is also what makes a loop read as "raspy".
- **Prop wash:** stronger lowpass = beaten-air whoosh; weaker lowpass (brighter)
  + louder mix = thin hiss. Small props are hissier/screechier.
- **Watch aliasing:** cap harmonic count so the top harmonic × max pitch stays
  under Nyquist (22050 Hz). Tiny loop uses 6 harmonics for this reason.

## Per-airframe acoustic profiles
Same drone code, different voice by size: `FPVDroneAudio.SetAirframeScale(visibleSize)`
sets `_tinyAirframe = visibleSize <= 0.35m`, chosen from the item's `VisibleSize`
at Attach time (before the startup chirp). The bomb drone (0.65m) keeps the 5"
buzz; the Tiny Drone (0.1625m) gets `BuildTinyLoopClip` plus higher-pitched
arming chirps (`_beepSource.pitch 1.5x`) and frame knocks. Clips are cached in
`static` fields, so each profile's loop is built once and reused across launches.

## Applies when
Any mod that needs a controllable engine/rotor/vehicle drone loop and wants to
avoid shipping/looping a recorded clip, or wants several sizes of the same
machine to sound distinct without new assets.
