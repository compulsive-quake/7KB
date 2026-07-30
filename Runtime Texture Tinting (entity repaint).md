# Runtime Texture Tinting (entity repaint)

Part of the [7DTD Modding Knowledgebase](README.md). How to recolor part of an entity's albedo textures at runtime (e.g. a vehicle "repaint" feature driven by a color picker), without shader/bundle changes, and without frame hitches while the user drags a color wheel. Reference implementation: Oppressor mod `src/EntityOppressor.cs` (dye section), July 2026.

---

## The problem

A live repaint that re-bakes tinted copies of the albedo textures on the CPU has two classic failure modes:

1. **Salt-and-pepper speckle** on materials that *almost* match the color mask. Hard per-pixel thresholds (e.g. `b > max(r,g) * 1.08`) sit right on top of JPEG/DXT compression noise: on the Oppressor, factory paint measured `b/max(r,g) ≥ 1.28` but the charcoal seat leather measured ~1.07 average — individual noisy pixels crossed 1.08 randomly, so ~37% of the seat got tinted as speckle.
2. **Massive drag lag**: `GetPixels()` (float `Color[]`, 67 MB for 2048²) + full-pixel loop + `SetPixels` + `new Texture2D` + `Destroy` per texture per color change = hundreds of ms per rebuild, several times a second while dragging.

## Mask: hue window × saturation gate × chroma gate (in HSV)

**RGB-ratio masks (blue dominance `b/maxRG`, separation `b-r`) fail on dark paint** — any brightness floor (`b <= 0.10`) or absolute-separation gate silently drops dark navy panels (V ≈ 0.1–0.2), which then stay factory-colored while the rest of the bike repaints (July 2026 Oppressor bug: fenders never changed). Work in HSV instead — hue is stable across brightness:

```csharp
// Measured (temp/cluster-stats.ps1): paint hue 204-209 S≈0.70; dark fender
// navy hue 222-230 S≈0.5 V≈0.2; seat weave hue 207 S≈0.16; silver metal
// S≈0.10; cyan dash screen hue 184 S≈0.47. Hue+sat separate ALL of them.
static float DyeMaskWeight(float hueDeg, float s, float v)
{
    float hueW = SmoothRange(192f, 200f, hueDeg)            // excludes cyan screen
               * (1f - SmoothRange(230f, 250f, hueDeg));    // excludes violet
    float satW = SmoothRange(0.25f, 0.45f, s);              // excludes seat + bare metal
    float chromaW = SmoothRange(0.025f, 0.06f, s * v);      // kills hue noise in near-blacks
    return hueW * satW * chromaW;
}
```

Key insight: on a factory-painted asset, *everything* bluish often shares one hue (Oppressor: seat weave, silver metal and paint all sit at hue ≈ 207) — **saturation, not hue, separates paint from trim**; hue separates paint from *other* saturated colors (the cyan screen, 20° away — use an asymmetric window). The soft weight is a lerp factor `lerp(original, tinted, w)` — boundary pixels blend instead of flickering. **Gotcha: Unity's `Mathf.SmoothStep(from,to,t)` is an interpolator, not GLSL `smoothstep(edge0,edge1,x)`** — write your own (`t=clamp01((x-e0)/(e1-e0)); t*t*(3-2t)`).

Validate offline before shipping, **against every albedo the model uses** — the original thresholds were only simulated on 2 of 5 textures, and the untested ones held the dark panels that broke. A PS 5.1 + System.Drawing script applying the exact mask/tint math to the source JPGs catches this in seconds (see Oppressor `temp/tint-sim3.ps1`, `temp/cluster-stats.ps1`, `temp/mask-viz.ps1`). PS 7 CAN `Add-Type` System.Drawing if you reference the whole forwarding chain: `-ReferencedAssemblies System.Drawing.Common, System.Drawing.Primitives, System.Private.Windows.GdiPlus, System.Private.Windows.Core` (add each assembly the compile error names next).

## Renderer exclusion lists go stale — audit them when the mask changes

A "skip these renderers entirely" list (added to protect a part the mask couldn't handle) **silently overrides every later mask improvement** for everything sharing that renderer. Oppressor July 2026 bug #2: `Cylinder009` was excluded wholesale to keep the old RGB-ratio mask off the cyan dash screen — but that one mesh also contained the entire nose-top cowl's navy panels, so after the HSV mask shipped (which excludes the screen *per-texel* via the hue window), the nose stayed factory-colored and nobody could tell why: the mask simulation showed those texels 100% caught. Per-texel gates beat per-renderer exclusions; when a mask upgrade makes an exclusion redundant, delete it.

**Mapping "which visible part is stuck" → "which texels":** Sketchfab-style COLLADA exports keep per-node `<instance_material>` bindings and single-indexed `<polylist>` (one index stream for position + UV), so a ~60-line script can rasterize any node's UV triangles over its albedo and print its world bbox (Oppressor `temp/uv-viz.ps1`, `temp/bbox-all.ps1`). Orient the model from part clusters (headlight barrels / dash / intake fan = front), then bbox → node → UV island tells you exactly which texture region a stuck part reads from — no guessing from texture thumbnails.

## Tint: replace hue, preserve the texel's own saturation and value

Mapping luma → `dye * shade` flattens every masked texel to the dye's exact hue/sat. On brushed metal (navy↔silver gradients) the mask cuts partway through the gradient and the flattened color bands against the untouched neighbors — reads as **streaky "corrupt" directional lines** in game. Instead re-hue in HSV, scaling the texel's own S and V against the factory paint's measured reference (Oppressor: S 0.70, V 0.30):

```csharp
shade = texelV / 0.30f; if (shade > 1) shade = 1 + (shade - 1) * 0.45f;  // highlight rolloff
newV = dyeV * shade;
newS = min(dyeS * (texelS / 0.70f), 1);
if (newV > 1) { newS *= 1 - min((newV - 1) * 0.6f, 0.6f); newV = 1; }    // whiten = desaturate
rgb = HSVToRGB(dyeH, newS, newV);
```

A typical panel comes out at exactly the picked color; half-saturated gradient texels come out half-saturated — navy→silver becomes red→silver smoothly. The tint now depends on two per-texel values, so the 1D shade LUT becomes a **2D (sat, val) LUT**: store S and V bytes per masked texel, LUT 64×128 (`(s>>2)*128 + (v>>1)`, 8 k entries, <1 ms rebuild per color change) — apply stays a lookup + byte-lerp.

## Performance: precompute + LUT + staged apply

One-time precompute per unique texture (trigger it when the picker UI opens, not on first drag):
- readable copy via `Graphics.Blit` → `ReadPixels` (bundle textures are not CPU-readable), `GetPixels32`, then `Destroy` the copy
- store: `int[] indexes` (masked texels only), `byte[] weights`, `byte[] shades` (shade*64, rolloff pre-applied), `Color32[] baseColors`
- create ONE persistent tinted `Texture2D(RGBA32, mipChain:true)` initialized to the base pixels; textures with no meaningful mask coverage are dropped entirely (never touched at runtime)
- copy `wrapMode`, `filterMode` **and `anisoLevel`** from the original — a fresh Texture2D defaults to aniso 1, which smears panels at glancing angles in a way the original (typically aniso 4-8) didn't

Per color change:
- build a **256-entry shade→Color32 LUT** for the new dye (tint depends only on shade, not the pixel)
- per texture: write `lerp(base, LUT[shade], weight)` in byte math over masked indexes **directly into `tex.GetPixelData<Color32>(0)`** (a `NativeArray` view of the CPU buffer — no SetPixels copy), then `Apply(updateMipmaps:false)` (mip-0 upload only)
- **round-robin: at most ONE texture per frame** — a few ms per frame instead of one big rebake
- once the color has been stable ~0.5 s, `Apply(updateMipmaps:true)` one texture per frame to regenerate mips (mips are stale-colored during the drag; unnoticeable because the camera is close while painting)

`GetPixelData<Color32>` requires the texture to actually be `TextureFormat.RGBA32` (raw byte view — `Color32` layout matches only RGBA32). Restore-to-factory = reassign the original bundle textures to the materials (keep them; `renderer.materials` instantiates per-entity material copies, safe to retarget). Destroy the tinted textures in `OnEntityUnload` — they are not parented to the entity GO and leak otherwise.

## Related

- [Paint & Textures](Paint%20&%20Textures.md) — the *block* paint system (texture IDs, unrelated mechanism)
- [Vehicles](Vehicles.md)
