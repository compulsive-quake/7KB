# Synthesized Drone Rotor Audio

How the FPV mod's fully-synthesized racing-quad sound works, what made early
versions sound wrong, and a repeatable workflow for tuning synthesized audio
against a real recording. Reference implementation: `FPV/src/FPVDroneAudio.cs`
(`BuildRacingLoopClip`, `UpdateFlightAudio`).

## The RPM → pitch model
- Render one seamless loop clip at a fixed reference RPM, then sweep
  `AudioSource.pitch = currentRpm / renderRpm` at play time. Never re-render
  per frame.
- **Unity clamps `AudioSource.pitch` to ±3** — pick the render RPM so the top
  of the sweep stays under 3. FPV: idle 3500 RPM → max 28000 RPM, rendered at
  10000 RPM ⇒ pitch 0.35..2.8.
- 5" racing quad ground truth (from analyzing a real recording): motors idle
  ~3500 RPM at zero throttle, cruise ~7500, full send ~28-33k. With 2-blade
  props the audible fundamental is the blade-pass rate = RPM/30 (≈117 Hz idle,
  ~250 Hz cruise, ~1.1 kHz flat out). Too-narrow a sweep (or too low a top
  end) reads as a massive heavy-lift drone, not a racer.
- Smooth commanded throttle into RPM with a ~0.08 s rise / 0.16 s fall
  exponential so pitch punches with the stick like real motors.
- **Spin-up: don't render a separate one-shot clip.** A dedicated spin-up
  sweep hardcodes a target RPM (wrong when launch context varies — pad idle
  vs scripted climb) and its synthesis drifts out of character whenever the
  loop is retuned. Instead spool the loop itself: on launch, ramp an
  `effectiveRpm = Lerp(~400, commandedRpm, SmoothStep(elapsed/spoolSeconds))`
  and multiply the loop volume by the ramp (~1.2 s reads like a real arm).
  Same timbre by construction, and it lands on whatever RPM the flight model
  is actually commanding.

## What causes "raspy" and how to avoid it
Spectral analysis of a real quad shows a **fundamental-dominant** tone: 2nd
harmonic ≈ 0.25-0.3 of the fundamental, everything above falls off fast.
Rasp came from three stacked sources:
1. Saw-like harmonic series (h2=0.62, h3=0.40, …). Fix: steep rolloff
   (1.0, 0.28, 0.12, 0.06, …).
2. Hot drive into a `tanh` soft-clipper — the saturation regenerates the
   harmonics you just removed. Keep the tonal bed well under the knee and let
   post-normalization restore level.
3. Differentiated-white-noise "prop wash" rises 6 dB/oct forever → tape hiss.
   One lowpass pole is NOT enough to beat the rising slope; cascade two
   one-pole LPs (coeff ≈ 0.3 at 44.1 kHz) so the noise peaks in the low kHz
   and falls off up top.
Keep the four-motor detune (±1-4%) — the beating between motors is the
signature "angry swarm" shimmer and shows up in real recordings as a cluster
of peaks around the fundamental.

## Tuning workflow: match a reference recording
ffmpeg + Python/numpy (both installed via chocolatey / C:\Python311):
1. `ffmpeg -i ref.mp3 -ac 1 -ar 44100 ref.wav`
2. Track the fundamental over time with a harmonic product spectrum (log-mag
   FFT summed at 1x,2x,3x,4x decimation) — the raw strongest peak is often a
   harmonic, not the fundamental.
3. Compare band-energy shares (e.g. 200-400 / 400-800 / … / 6.4k-12k Hz)
   between reference and a numpy re-implementation of the synth; iterate
   filter/level constants in Python (seconds per iteration), then port the
   final constants to C#. Account for the play-time pitch multiplier when
   comparing bands — the rendered clip shifts up in flight.

## Onboard-mic / remote-mic gain pattern
FPV streams drone audio to the operator ("remote microphone"): sources flip to
`spatialBlend = 0` while in the feed and every onboard volume is multiplied by
a user gain (Options → HUD → AUDIO: enable toggle + volume slider, persisted
in `controls.json` as `RemoteMicEnabled`/`RemoteMicVolume`). Gain is read from
config every frame for looping sources, and applied at `Play()` time for
one-shots, so slider changes are live. Mic off mutes only the feed — the
world-spatialized sound bystanders hear is untouched.

## Applies when
Any mod synthesizing engine/rotor/machine loops in code, or pitch-sweeping a
single rendered clip across an RPM range. See
[Runtime WAV Loading (AudioClip from disk)](<Runtime WAV Loading (AudioClip from disk).md>)
for the sampled-audio side of the same pipeline.
