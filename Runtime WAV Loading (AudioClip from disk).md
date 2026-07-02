# Runtime WAV Loading (AudioClip from disk)

When a code-only 7DTD mod needs a sampled sound but can't rebuild its Unity
asset bundle (`.unity3d`), you can ship a loose `.wav` in the mod's `Resources/`
folder and decode it into an `AudioClip` at runtime — no bundle, no
`UnityWebRequest`.

## How
1. Ship the WAV in `resources/` and copy it to `<ModDest>/Resources/` in
   `deploy.ps1` (same pattern as the goggles PNG in the FPV mod).
2. Resolve the deployed path via the mod instance path: `ModInfo`/`Mod.Path`
   (FPV exposes it as `FPVModInit.ModPath`), then
   `Path.Combine(modPath, "Resources", "file.wav")`.
3. `File.ReadAllBytes(path)` → parse the RIFF/WAVE header → decode PCM into a
   `float[]` normalized to -1..1 → `AudioClip.Create(name, samplesPerChannel,
   channels, sampleRate, false)` then `clip.SetData(data, 0)`.

## WAV parsing notes
- Walk the chunk list (`4-byte id` + `4-byte little-endian size` + payload);
  don't assume `fmt ` is immediately followed by `data` — extra chunks (LIST,
  etc.) can sit between them. Chunks are word-aligned: an odd size has a pad byte.
- `fmt ` fields: audioFormat(2) must be 1 (uncompressed PCM), channels(2),
  sampleRate(4), then bitsPerSample at offset +14.
- Decode per bit depth: 8-bit is **unsigned** (0..255, centre 128); 16/24/32-bit
  are signed little-endian. 24-bit needs manual sign extension.
- `AudioClip.Create` takes **samples per channel** (not total interleaved
  samples); `SetData` takes the full interleaved `float[]`.
- Reference implementation: `FPV/src/FPVWavLoader.cs`.

## Applies when
Any mod that wants recorded audio without an asset-bundle round-trip. Same
runtime-`AudioClip` path the FPV mod already uses for its synthesized rotor
sounds — this just fills the buffer from a file instead of a synth.
