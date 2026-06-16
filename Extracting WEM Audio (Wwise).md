# Extracting WEM Audio (Wwise / Helldivers 2 mods)

How to pull playable audio out of Wwise game-audio files — useful for sourcing
sound assets (engine loops, weapon SFX) to ship in a 7DTD mod. Verified against
Helldivers 2 sound-mod files (June 2026).

## What the files are

Helldivers 2 audio mods ship as Autodesk **Stingray** archives:

- `<hash>.patch_0` — an archive containing a Wwise **SoundBank** (`BKHD` magic)
  plus embedded **WEM** audio (each a `RIFF....WAVE` block).
- `<hash>.patch_0.stream` — a sibling container holding the **full-quality
  streamed** WEMs as their own complete `RIFF/WAVE` files. The `.patch_0` only
  carries a short ~1s **prefetch** copy of each streamed sound; the real audio is
  in `.stream`.
- `manifest.json` — mod metadata (option names, descriptions, icons).

WEM codec is **Wwise Vorbis** (`fmt ` chunk format code `0xFFFF`). **ffmpeg
cannot decode this** — you need vgmstream (or ww2ogg + revorb).

## Tooling

- **vgmstream-cli** — the one decoder that handles Wwise Vorbis WEM directly.
  Download the Windows build:
  `https://github.com/vgmstream/vgmstream/releases/latest/download/vgmstream-win64.zip`
  Decode: `vgmstream-cli.exe -o out.wav in.wem`
- ffmpeg/ffprobe — only for inspecting/converting the resulting WAVs.

Gotcha: vgmstream-cli is a Windows exe — give it **real Windows paths**. A
cygwin/Git-Bash `/tmp/...` path makes it print `failed opening <file>`.

## Extraction recipe

1. **Carve** every WEM from both `.patch_0` and `.patch_0.stream` by scanning for
   `RIFF`, reading the little-endian size at offset+4, and slicing
   `data[i : i+4+size]` (skip any slice that runs past EOF — those are truncated
   prefetch entries whose data lives in `.stream`).
2. **Decode** each carved `.wem` to `.wav` with vgmstream-cli; treat a
   `returncode==0` + `>1KB` WAV as success.
3. The `.stream` WEMs are self-contained complete RIFFs — **no header stitching
   needed**. (Stitching the bank's prefetch header onto raw `.stream` bytes does
   NOT work — vgmstream rejects it. Just carve the `.stream` directly.)
4. **Deduplicate** by content hash: a single bank references the same sound many
   times, so 386 raw WEMs collapsed to ~339 unique WAVs.

The full-quality loops (e.g. a 19.35s engine loop) come from the `.stream`
container; the in-bank copies of those same sounds are only ~1.3s prefetch.

## Don't commit the output

Extracted WAVs are large (278 MB for one mod) and vgmstream is a third-party
binary — gitignore both rather than committing. Also: extracted content is
someone else's mod asset; fine for personal reference, mind licensing before
shipping any of it.
