# Unity DDS Import Quirks (AC kn5 textures)

Assetto Corsa kn5 files embed DDS textures in layouts Unity's asset importer
refuses. Every rejected file logs an importer `LogError`, and AssettoCar's
headless build (`AssettoHeadlessBuild.Run`) fails on ANY error — so one quirky
texture used to kill the whole conversion. All repairs live in
`AssettoCar/src-tauri/resources/unity-template/Assets/Editor/AssettoCarBuilder.cs`
(`FixDdsForUnity` + `DecompressBcDds`).

What Unity's DDS importer accepts: DXT1/3/5 (BC1/2/3) and BGRA-masked
uncompressed, with mip chains either absent or complete down to 1×1.

AC cars routinely ship, and the builder now repairs:

| AC layout | Repair |
| --- | --- |
| RGBA-masked uncompressed (R/B swapped) | swap R/B bytes, patch masks |
| L8 / A8L8 luminance | expand to A8R8G8B8 |
| Truncated mip chains | drop to top level, clear mip flags |
| **DXT1/3/5 with non-multiple-of-4 width/height** (gauge faces, needles, badge plaques — e.g. 380×379, 14×35) | CPU-decode BC blocks → uncompressed A8R8G8B8 at the ORIGINAL size (added 2026-07-15; 11 such textures in the McLaren 765LT killed the build with 22 errors) |

Gotchas:

- Do NOT "fix" odd-sized BC textures by relabeling the header to the padded
  block grid (e.g. 14×35 → 16×36): the physical data does cover full blocks,
  but UVs span the padded size afterwards — a 14-px-wide needle stretches ~13%.
  Decode instead.
- DXT1 with `color0 <= color1` is 3-color + punch-through alpha mode; DXT3/5
  color blocks are always 4-color regardless.
- Unknown FourCCs (BC4/BC5/BC7/DX10 headers) are still skipped with a warning —
  candidates for the same decode treatment if a car ever needs them.
- The failing-build tell in `unity-work/Logs/headless.log`:
  `Invalid dimensions: ... must be a multiple of the formats (DXT5|BC3) block
  dimensions (4x4)` followed by `Unsupported Assets/Job/Textures/<x>.dds file.`

## DDS goes through IHVImageFormatImporter, not TextureImporter

`.dds` files are imported by `IHVImageFormatImporter` (2026-07-15, confirmed by
the generated `.meta` files). Consequences:

- `AssetPostprocessor.OnPreprocessTexture` NEVER fires for them — the
  `normal_maps.txt` preprocessor in `AssettoTexturePreprocessor.cs` only
  affects PNG/JPG textures. DDS normal maps import as plain sRGB textures
  (Standard's `UnpackNormalmapRGorAG` still decodes RGB normals via the
  `x*a` trick, so lighting is roughly right, but they're not true NormalMap
  imports).
- Unity never (re)generates mipmaps for DDS: the file's own mip chain is all
  you get. Mip chains stripped by `FixDdsForUnity` stay mip-less in the
  bundle → shimmering/aliasing when minified.
- No aniso control: the importer API exposes only isReadable/filterMode/
  wrapMode/sRGB. DDS textures ship with `aniso: 1`.
- Pixel data is used as-is (uncompressed stays uncompressed — good for AC's
  deliberately-uncompressed AO/seam skins like `EXT_skin.dds`).

Generated paint textures (`__paint_baked.png`, `__paint_ao.png`) are PNG
precisely so they DO go through TextureImporter — the preprocessor gives all
Job PNG/JPGs `anisoLevel = 8` and gives the two paint bakes
`CompressedHQ` (BC7): default-quality DXT1 turns thin dark panel-gap lines
over flat paint into blocky crawl.

## AC paint material picking: int_/ext_ twins

Cars commonly ship interior twins of the body paint material (`int_carpaint`
alongside `ext_carpaint` — McLaren 765LT). Substring matching on "carpaint"
can latch onto the interior one, silently wiring the whole dye/livery/bake
system to the cabin (report.json said `paintMaterial = int_carpaint`; the
factory bake came out as the INT_OCC interior AO atlas multiplied by
`black.dds`). `PickPaintMaterial` now drops interior-named hits
(`int*`/`_int`/`interior`/`inner`) when an exterior hit exists and breaks
ties by triangle surface area. Check `Build/report.json` →
`paintMaterial`/`paintTexture` when a conversion's paint looks wrong.

## Smuggling an extra texture to the runtime via the detail slot

The paint material's `_DetailAlbedoMap` is set to the normalized AO bake
WITHOUT enabling `_DETAIL_MULX2` — the texture renders as nothing but is
serialized into the bundle, and the runtime (`AssettoPaint.NeutralAlbedo`)
fetches it with `mat.GetTexture("_DetailAlbedoMap")` to use as the albedo in
solid-color mode (white albedo × _Color used to flatten all panel-seam AO).
