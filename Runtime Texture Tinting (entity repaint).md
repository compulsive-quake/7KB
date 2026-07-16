# Runtime Texture Tinting (entity repaint)

Part of the [7DTD Modding Knowledgebase](README.md). How to recolor part of an entity's albedo textures at runtime (e.g. a vehicle "repaint" feature driven by a color picker), without shader/bundle changes, and without frame hitches while the user drags a color wheel. Reference implementation: Oppressor mod `src/EntityOppressor.cs` (dye section), July 2026.

---

## The problem

A live repaint that re-bakes tinted copies of the albedo textures on the CPU has two classic failure modes:

1. **Salt-and-pepper speckle** on materials that *almost* match the color mask. Hard per-pixel thresholds (e.g. `b > max(r,g) * 1.08`) sit right on top of JPEG/DXT compression noise: on the Oppressor, factory paint measured `b/max(r,g) ≥ 1.28` but the charcoal seat leather measured ~1.07 average — individual noisy pixels crossed 1.08 randomly, so ~37% of the seat got tinted as speckle.
2. **Massive drag lag**: `GetPixels()` (float `Color[]`, 67 MB for 2048²) + full-pixel loop + `SetPixels` + `new Texture2D` + `Destroy` per texture per color change = hundreds of ms per rebuild, several times a second while dragging.

## Mask: soft and double-gated

Measure the actual textures first (pull them from the Unity project and histogram candidate metrics per region). Then gate on **two independent metrics**, each smoothstepped, and multiply:

```csharp
// factory paint: b/maxRG >= ~1.28, b-r >= ~0.14
// seat leather:  b/maxRG ~= 1.07, b-r ~= 0.04
static float DyeMaskWeight(float r, float g, float b)
{
    if (b <= 0.10f) return 0f;
    float maxRG = Mathf.Max(Mathf.Max(r, g), 0.001f);
    return SmoothRange(1.14f, 1.32f, b / maxRG) * SmoothRange(0.055f, 0.115f, b - r);
}
```

The soft weight is a lerp factor `lerp(original, tinted, w)` — boundary pixels blend instead of flickering. **Gotcha: Unity's `Mathf.SmoothStep(from,to,t)` is an interpolator, not GLSL `smoothstep(edge0,edge1,x)`** — write your own (`t=clamp01((x-e0)/(e1-e0)); t*t*(3-2t)`).

Validate thresholds offline before shipping: a PS 5.1 + System.Drawing script that applies the exact mask/tint math to the source JPGs and writes PNGs catches bad thresholds in seconds (see Oppressor `temp/tint-sim.ps1`).

## Tint: normalize shade to the paint's own brightness

`tint = dye * shade` where `shade = pixelLuma / paintRefLuma` (measured average luma of the factory paint, ~0.30 on the Oppressor). Then a typical panel comes out at exactly the picked color. Avoid `clamp01(shade * boost)` — it clips bright panels per-channel into a flat neon sheet with hue shift. Instead:

- compress shade above 1: `shade = 1 + (shade-1) * 0.45`
- if a channel would clip: renormalize by the max channel (keeps hue), then push toward white proportional to the excess (reads as specular highlight).

## Performance: precompute + LUT + staged apply

One-time precompute per unique texture (trigger it when the picker UI opens, not on first drag):
- readable copy via `Graphics.Blit` → `ReadPixels` (bundle textures are not CPU-readable), `GetPixels32`, then `Destroy` the copy
- store: `int[] indexes` (masked texels only), `byte[] weights`, `byte[] shades` (shade*64, rolloff pre-applied), `Color32[] baseColors`
- create ONE persistent tinted `Texture2D(RGBA32, mipChain:true)` initialized to the base pixels; textures with no meaningful mask coverage are dropped entirely (never touched at runtime)

Per color change:
- build a **256-entry shade→Color32 LUT** for the new dye (tint depends only on shade, not the pixel)
- per texture: write `lerp(base, LUT[shade], weight)` in byte math over masked indexes **directly into `tex.GetPixelData<Color32>(0)`** (a `NativeArray` view of the CPU buffer — no SetPixels copy), then `Apply(updateMipmaps:false)` (mip-0 upload only)
- **round-robin: at most ONE texture per frame** — a few ms per frame instead of one big rebake
- once the color has been stable ~0.5 s, `Apply(updateMipmaps:true)` one texture per frame to regenerate mips (mips are stale-colored during the drag; unnoticeable because the camera is close while painting)

`GetPixelData<Color32>` requires the texture to actually be `TextureFormat.RGBA32` (raw byte view — `Color32` layout matches only RGBA32). Restore-to-factory = reassign the original bundle textures to the materials (keep them; `renderer.materials` instantiates per-entity material copies, safe to retarget). Destroy the tinted textures in `OnEntityUnload` — they are not parented to the entity GO and leak otherwise.

## Related

- [Paint & Textures](Paint%20&%20Textures.md) — the *block* paint system (texture IDs, unrelated mechanism)
- [Vehicles](Vehicles.md)
