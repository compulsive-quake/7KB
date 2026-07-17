# Synthesized Drone Rotor Audio

How the FPV mod's racing-quad sound works, what made early versions sound wrong,
and a repeatable workflow for tuning it against a real recording. Reference
implementation: `FPV/src/FPVDroneAudio.cs` (`BuildRacingLoopClip`,
`UpdateFlightAudio`).

**Status (2026-07-16):** the bomb drone now flies on a *recorded* loop authored
by `FPV/tools/make_engine_loop.py` — see "Authoring a recorded loop" below. The
synth is kept as the missing-WAV fallback, and the tiny whoop is still fully
synthesized. The RPM→pitch model below is shared by both: a recorded loop is a
drop-in for a synth loop **only** if it is resampled so its own blade-pass sits
exactly at the reference RPM.

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

## Authoring a recorded loop for an RPM-driven pitch model
A raw excerpt of a real flight cannot be used as a loop. An earlier FPV attempt
was abandoned as "mushy pitch response"; the recording wasn't the problem, the
missing prep was. Three things must be fixed, in this order
(`FPV/tools/make_engine_loop.py` does all three):

1. **Pitch-flatten to the reference RPM.** A real drone maneuvers, so its RPM
   wanders (the FPV source swept 250→320 Hz blade-pass). `pitch = rpm/renderRpm`
   is only correct if the clip's own blade-pass IS `renderRpm`. Fix: variable-
   rate resample — walk the input at rate `target/f0(t)` using the tracked f0
   contour, which both removes the wander and lands the tone on target.
   - **Close the loop on the result.** Output pitch scales as `1/corr`, and any
     f0 tracker is biased (smoothing lag), so measure the rendered fundamental
     and iterate `corr *= measured/target`. FPV converged in 1 iteration to
     0.03%. Getting this ratio inverted makes it diverge — the sign is not
     obvious, so always print the error per iteration.
2. **Envelope-flatten, or the loop has a *rhythm*.** The excerpt's own loudness
   and brightness swell over its length; looping replays that swell every loop
   period. Perceptually this is a "WRREEERRREEE" pulse instead of a steady
   "WRRRRR", and it is the single most audible loop artifact. Fix: STFT, divide
   each bin's magnitude by its own **slow** envelope (lowpass ≈14 Hz), keep the
   phases. Variation faster than the cutoff (noise texture, motor-beat shimmer)
   survives and is what still makes it sound real. FPV: loudness wobble
   6.16%→0.92%, brightness wobble 8.01%→0.33%.
   - Choose the STFT hop so `loopLen % hop == 0`, and compute the envelope on a
     3× tiled copy taking the middle — then the gain is itself periodic and the
     loop stays seamless.
3. **Loop seamlessly by construction, not by fading.** Pick `renderRpm` so the
   blade-pass is an integer number of samples per cycle (FPV: 10000 RPM ⇒
   333.33 Hz ⇒ exactly 144 samples at 48 kHz), then cut a whole number of
   cycles. The harmonics wrap phase-exactly with no fade. Only the noise floor
   needs a crossfade: blend `y[N..N+F]` over `y[0..F]`, which leaves
   `out[0] = y[N]` following `out[-1] = y[N-1]` — adjacent source samples, so
   continuity is exact.

**Pick the region by spectral stationarity, not pitch stability.** Pitch is
fixable (step 1); timbre and loudness wander are not. FPV's first pick was the
most pitch-stable window and ranked only 92/1082 on stationarity; switching to
the most stationary window cut loudness wobble ~30% before any other processing.
Score candidates on per-frame loudness-normalized spectral shape deviation +
loudness variance.

**Measurement gotcha:** a periodic signal's envelope spectrum is *necessarily* a
line spectrum at multiples of the loop rate, so "modulation prominence at the
loop rate" is high no matter how smooth it sounds — a useless metric that reads
as a real defect. Measure modulation **depth** instead: `std(envelope lowpassed
below ~6 Hz) / mean(envelope)`, filtered circularly. Sanity-check any audio
metric against a known-bad and known-good version before trusting it.

Also high-pass (~130 Hz) the wind rumble out of an outdoor recording if the mod
layers its own wind, and remember hardcoded resource lists: FPV's `deploy.ps1`
copies an explicit WAV whitelist, so a new file silently never ships and the
code falls back to the synth with no error.

## Onboard-mic / remote-mic gain pattern
FPV streams drone audio to the operator ("remote microphone"): sources flip to
`spatialBlend = 0` while in the feed and every onboard volume is multiplied by
a user gain (Options → HUD → AUDIO: enable toggle + volume slider, persisted
in `controls.json` as `RemoteMicEnabled`/`RemoteMicVolume`). Gain is read from
config every frame for looping sources, and applied at `Play()` time for
one-shots, so slider changes are live. Mic off mutes only the feed — the
world-spatialized sound bystanders hear is untouched.

## Applies when
Any mod synthesizing engine/rotor/machine loops in code, pitch-sweeping a single
rendered clip across an RPM range, or turning a field recording of a real engine
into a loopable RPM-driven asset. See
[Runtime WAV Loading (AudioClip from disk)](<Runtime WAV Loading (AudioClip from disk).md>)
for the sampled-audio side of the same pipeline.
