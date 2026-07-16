# World-Space Block UI (look-at buttons)

Part of the [7DTD Modding Knowledgebase](README.md). How to replace a
model-entity block's look with a code-built world-space "screen" (buttons,
labels, highlights) and let the player interact with **sub-regions of one
block** — look at a specific button, tap E, get that button's action. No asset
bundle, no physics raycasts, no NGUI. Reference implementation: Elevator
`src/ElevatorPanelModel.cs` + `src/ElevatorPanelArt.cs` +
`src/ElevatorPanelView.cs` (+ `BlockElevatorPanel.cs`) — the cab panel that
draws one round button per floor and rides E-presses straight to the floor the
player is looking at.

---

## Own the model, don't re-dress someone else's (preferred)

**Do this instead of the hide-the-vanilla-renderers approach below.** You can
give a block a model you build entirely in code, with no Unity project and no
asset bundle:

1. Define the block from scratch in `blocks.xml` with `Shape="ModelEntity"`
   and `Model="<YourMod>/<yourModelName>"` — a path no bundle provides.
   `ModelOffset` / `MultiBlockDim` / `Place` copied from whatever vanilla
   block you want to mount like.
2. Harmony-patch the **private** `BlockShapeModelEntity.getPrefab()` (target it
   by string) and return your own `Transform` when
   `__instance.modelName == "<yourModelName>"`. `modelName` is genuinely
   public and is `GameIO.GetFilenameFromPathWithoutExtension(Model)`, assigned
   in `Init` before anything calls `getPrefab`. Everything else falls through
   to the vanilla loader.

That one interception covers **every** path that resolves a block model:
`Init` → `GameObjectPool.AddPooledObject` (which calls its load callback
*immediately*, so your builder runs during blocks.xml load — main thread,
`Mod.Path` already set), the block entity itself, and
`ItemClassBlock.CreateMesh` → `CloneModel` (the held-item **placement
preview** and a mover's **proxy clone**).

Why this beats re-dressing: the engine's force-enable of every renderer under
the anchor is now *correct* (they're all yours), pooling recycles your object,
there's no vanilla material to go magenta, and `MeasureVanilla`-style
footprint probing disappears — you authored the geometry, so you know it.

**Three traps:**

- **Inactive prefab ⇒ invisible preview and proxy.** `Instantiate` copies
  `activeSelf`, and only the chunk path re-activates
  (`Chunk`: `objectForType.SetActive(true)`); `PoolItem.Instantiate()` and
  `CloneModel` do **not**. A prefab you `SetActive(false)` to stop it
  rendering at world origin will place fine but preview and ride invisible.
  Park an **active** prefab under an **inactive holder** GameObject:
  `activeInHierarchy` is false for the template, every clone comes out active.
- **Never return null from `getPrefab`.** `PoolLoadCallback` throws on null,
  which kills blocks.xml loading for the whole game. Return a bare named
  GameObject and log instead.
- **A collider on the model root silently rewrites the block's bounds.**
  `PoolCreateOnceToAllCallBack` reads a root `BoxCollider`/`CapsuleCollider`
  into `isCustomBounds`, after which `GetBounds` = `rotation * colliderBounds
  + ModelOffset + (0.5, 0, 0.5)`. With a `ModelOffset` like `-.25,0,0.5`
  that lands the hit box a block off the art. Omit the collider and the block
  keeps `BlockShape`'s default per-voxel bounds (block-level focus and
  activation still work — they're voxel raycasts, not physics).

If the patch ever stops applying, `DataLoader.LoadAsset` fails and
`getPrefab` falls back to `@:Entities/Misc/block_missingPrefab.prefab` — the
**magenta box**. A pink block is the first symptom of a broken model patch.

**...but it is not the only cause of a pink block, and the other one is
sneakier.** If breaking *one* block turns the *others* magenta (and the next
one placed comes up blank white), the model patch is fine — the pool destroyed
your shared materials. `GameObjectPool.DestroyObject` frees every runtime-
created material under a pooled clone, and `HideAndDontSave` does not stop it.
A code-built model **must** ship the `DestroyObject` exemption patch alongside
the `getPrefab` one; see
[Runtime Procedural VFX](Runtime%20Procedural%20VFX.md) §"`HideAndDontSave`
does NOT save a shared material from `GameObjectPool`" for the mechanism and
the fix. Rule of thumb: *your* block goes pink → suspect `getPrefab`;
*neighbouring* blocks go pink after a destroy → suspect `DestroyObject`.

Splitting static from dynamic: put the unchanging chassis in the prefab (it is
shared by every instance *and* is what the placement preview shows), and let a
per-block view add only the changing children onto each clone. The pooling
death-check below still applies to those children.

## Re-dressing a model-entity block at runtime (legacy — prefer the above)

This is how Elevator's panel used to work, and why it was abandoned: hiding
another model's renderers is an unwinnable arms race with the engine, and the
failure modes are exactly "the vanilla model flickers back" and
"it went white/pink". Kept because it's still the only option when you must
alter a block you don't define (a vanilla block you can't replace).

Get the block's live model transform from
`world.ChunkCache.GetBlockEntity(masterPos)` (`bHasTransform` /
`bed.transform`), **disable every `Renderer` under it** (colliders stay, so
focus + activation keep working), and parent your own meshes to it. Parenting
buys you the block's placement rotation, any `ModelOffset` (including custom
offset variants), and the floating origin for free.

**Lifecycle gotcha — the model transform is POOLED, not destroyed.** Two
engine behaviors (confirmed in decompiled `Chunk`, 7DTD 3.x) break the naive
"rig dies with its transform" pattern:

1. **Chunk unload/hide pools the model.** `Chunk.poolBlockEntityTransform`
   first re-enables *all* renderers, then hands the GameObject to
   `GameObjectPool.PoolObject`, which deactivates and reparents it **with
   children intact** — your rig rides into the pool. `rig.Root == null` never
   fires, so the stale rig keeps "working" against a pooled object, and the
   pool later hands that object (rig still attached) to whatever block of the
   same model type needs one next. Symptoms: custom face vanishes / vanilla
   model back after revisiting an area; ghost faces on unrelated blocks.
   Correct death check, per frame: rig is dead unless
   `bed = ChunkCache.GetBlockEntity(pos)` exists, `bed.bHasTransform`, and
   `bed.transform == rig.Anchor` (reference compare). On death,
   `Object.Destroy(rig.Root)` explicitly — it may be living inside the pool.

2. **Renderer hides get stomped.** `Chunk.SetBlockEntityRendering` (chunk
   redisplay, render toggles) does `GetComponentsInChildren<Renderer>` and
   force-sets `.enabled` on every one — vanilla *and* yours. Hiding the
   vanilla renderers once is not enough: cache the vanilla `Renderer[]` at
   rig build (before adding your own meshes, so the capture is purely
   vanilla) and re-assert `enabled = false` every frame. Your own renderers
   need no re-assert — the same walk re-enables them symmetrically.

If a block mover (see [Moving Blocks at Runtime](Moving%20Blocks%20at%20Runtime%20(SetBlocksRPC).md))
carries the block, its proxy clone is a **fresh prefab instantiate** and
reverts to the vanilla look — add a dress-the-clone hook right after the
proxy's `ItemClassBlock.CreateMesh` call.

**Wrapper-transform trap (confirmed in decompile, 3.0.x):** the two paths
hand you *different* transforms. The chunk path's `bed.transform` **is** the
model clone — `Chunk` places it with `shape.GetRotation(bv)` + the rotated
`ModelOffset`. But `ItemClassBlock.CreateMesh(Local)` first creates a **bare
wrapper `GameObject`** under your parent and calls
`BlockShapeModelEntity.CloneModel` into it — the rotated clone is the
wrapper's child, and the wrapper itself stays identity. Anchor your rig to
`wrapper.GetChild(0)`, not the wrapper: on the wrapper the rig renders
world-axis-aligned, so a rotated block's face rides mirrored (180° mounts —
you see the double-sided quads from behind) or facing the wrong wall (90°
mounts), then snaps right on landing. The clone's frame matches
`bed.transform` exactly (both end up at `blockPos + rotatedOffset +
(0.5, 0, 0.5)`), so a rig built for the chunk path works unchanged.

Rebuild-on-change: hash the displayed state (names, current floor, blocked
flags…) each frame and rebuild the rig's children only when the hash moves;
per-frame work stays trivial. Anything that **animates** (a hover highlight, a
blinking destination lamp) must NOT be driven through the hash — bake only the
*event* state into it (e.g. which floor is the blink target, from
`ElevatorMover.TargetFloor`) so the rig rebuilds once when that changes, then
keep the animated quad as a persistent child and toggle/move it every frame
from `Update` (`blink.SetActive(Mathf.Repeat(Time.time, period) < duty)`).
Fold the pulse into the hash instead and you rebuild the whole rig every frame.
Reference: `ElevatorPanelView.UpdateBlink` (destination pulse) and
`UpdateHover` (aim highlight) — both walk the live rigs, not a rebuild.

## The wall-panel local frame (and the mirror trap)

Vanilla wall-mounted panel prefabs (`elevatorInsidePanelPrefab`,
`elevatorOutsideButtonPanelPrefab`; both `MultiBlockDim 1,2,1`) share this
model-transform frame:

- +Y up, **+Z points out of the wall** toward the viewer
- therefore the **viewer's right is the panel's −X** (Unity left-handed):
  everything you draw on the face is mirrored unless you compensate.

**The origin is NOT the block's bottom centre.** These blocks carry a
`ModelOffset` (`elevatorInsidePanel` is `-.25,0,0.5` — a quarter block
toward the viewer's right, half a block out to the wall face), and the
prefab's geometry compensates internally so the visible panel still sits
inside the block. Art drawn centred on the transform origin therefore lands
0.25 m right of the block and (at 1 m wide) pokes outside it.

**Ground truth** (measured from `doors.bundle` with UnityPy) — use these
numbers directly when authoring your own model to mount like vanilla's:

| prefab LOD0 | x | y | z |
|---|---|---|---|
| `elevatorInsidePanelPrefab` | 0 .. 0.483 (centre +0.242) | 0 .. 2.003 | −0.025 .. 0.018 |
| `elevatorOutsideButtonPanelPrefab` | ±0.079 | 1.055 .. 1.377 | −0.003 .. 0.031 |

The inside panel is half a block wide, the full 2 blocks tall, ~4 cm thick,
centred x +0.242 so the −0.25 `ModelOffset` lands it dead centre. **Note the
plate straddles z=0** rather than starting at it — authoring it `0..0.04`
stands the panel ~2 cm proud of the wall it should sit flush against.
Elevator's `ElevatorPanelModel` bakes these in as `PanelW/PanelCX/BackZ/FaceZ`.

If you must probe a *vanilla* model's footprint at runtime instead (legacy
re-dressing path only), union the `sharedMesh.bounds` of the **LODGroup's LOD0
renderers** mapped through `anchor.worldToLocalMatrix *
renderer.localToWorldMatrix` (NOT `Renderer.bounds`, a world AABB that lies on
rotated mounts). **LOD0 only, not all renderers**: these prefabs carry helper
children beside the visible model — `elevatorInsidePanelPrefab` ships an
animated `trap_arrow_indicator` 0.33 m off the wall and 0.38 m below the block,
and a union over `GetComponentsInChildren<Renderer>` swallows it, turning the
"panel" into a deep box.

Two compensations:

- **Textured quads**: run the U coordinate toward −X (u=0 at +X).
- **TextMesh**: a TextMesh reads correctly from its **−Z side**, so yaw it
  180° (`Quaternion.Euler(0,180,0)`). The flip also maps the glyph run
  direction onto the viewer's right, so `TextAnchor.MiddleLeft` text extends
  the way the viewer expects.

## Look-at sub-block interaction (no physics)

7DTD gives you block-level focus only, but `GetActivationText` is polled
every frame while the crosshair is on the block — so you can compute *which
part* of the block is aimed at yourself and both narrate it and act on it:

1. Ray = `player.cameraTransform` position/forward
   (`ElevatorBoundsRenderer.PlayerCamera()`).
2. Transform into the rig's local space (`InverseTransformPoint/Direction`),
   intersect with the face plane `z = FaceZ`, require the camera on the +Z
   side and a hit within reach (~4 m; actual activation range is still
   enforced by the game's own focus).
3. Map local (x, y) to a button row/cell; return -1 outside the bands.

Then:

- `GetActivationText` → "Press E to go to Floor 3 — Lab" for the aimed row
  (recomputed every frame, so it tracks the crosshair).
- `OnBlockActivated` → resolve the aimed row again and run its action; fall
  back to the old window/behaviour when no button is aimed.
- Hover feedback: per-frame, move a highlight quad onto the aimed row (pick
  the nearest rig when the ray crosses several panels).

Remember multiblock children: resolve `_blockPos` to the master
(`multiBlockPos.GetParentPos`) before keying any lookups — the plane math is
identical either way since it works in model space.

## Depth-tested world-space text (TextMesh without ZTest-Always)

The built-in font material (GUI text shader) is **ZTest Always** — TextMesh
labels shine through walls (fine for floating markers, wrong for text glued
to a surface). Fix: give the TextMesh renderer a `Sprites/Default` material
with the **font atlas as its texture**. The font atlas is alpha-only (samples
white + alpha), and TextMesh writes its `color` into vertex colours, which
`Sprites/Default` multiplies — so glyphs render tinted, alpha-blended, and
properly occluded.

```csharp
Font font = Resources.GetBuiltinResource<Font>("LegacyRuntime.ttf"); // 2022 name
Material textMat = new Material(Shader.Find("Sprites/Default")) { renderQueue = 3005 };
textMat.hideFlags = HideFlags.HideAndDontSave;
textMat.mainTexture = font.material.mainTexture;
// tm.font = font; renderer.sharedMaterial = textMat;
```

Dynamic fonts regrow their atlas and may swap textures — resync on
`Font.textureRebuilt` or glyphs go garbage after enough unique characters:

```csharp
Font.textureRebuilt += f => { if (f == font) textMat.mainTexture = f.material.mainTexture; };
```

TextMesh size math: world line height ≈ `fontSize`-independent
`6.4 × characterSize` (at fontSize 64) — pick `characterSize = targetHeight/6.4`.

TextMesh has no outline style. Fake one with four extra TextMesh copies at
diagonal offsets `±0.25 × characterSize`, coloured dark, placed slightly
behind the fill (~half a z-layer) on a material whose render queue is one
below the fill's — the queue split keeps rim under fill regardless of view
angle. `FontStyle.Bold` on the fill helps legibility at distance. Cost is
5 meshes per label; fine when rigs rebuild only on content-hash changes.

## Layering flat art without z-fighting

Coplanar transparent quads (plate → buttons → glow → text) sort by material
`renderQueue`, not reliably by distance — give every layer its own material
with an ascending queue (3000, 3001, …) *and* a small physical z step
(~0.003 m). Both together are flicker-free at all angles.

Reuse the mod's own XUi sprite PNGs (`UIAtlases/UIAtlas/*.png`) as the quad
textures — the world panel and the 2-D window then share one look by
construction. Loading them needs `Texture2D.LoadImage` +
`UnityEngine.ImageConversionModule` reference + `Mod.Path`, and every cached
texture/material needs `HideAndDontSave` — all covered in
[Runtime Procedural VFX](Runtime%20Procedural%20VFX.md).

**Regenerating the sprites, and the deploy kind.** Elevator's plate/button
PNGs are generated art, not photos: `tools/gen_ui_sprites.ps1` draws them
procedurally (fixed `Random(42)` seed, so re-runs are byte-stable) and writes
`UIAtlases/UIAtlas/*.png`. Run it under **Windows PowerShell 5.1**
(`powershell.exe`), not pwsh 7 — the script's `System.Drawing` `Add-Type`
needs .NET Framework GDI+; pwsh's `System.Drawing.Common` drags in private
GdiPlus assemblies `Add-Type` can't reference. The `Panel()` function is the
brushed-metal plate (bright satin aluminium: low-amplitude per-row streaks =
horizontal brush lines, a broad off-centre sheen, a gentle top-lit vertical
gradient, faint cool tint); `elev_panel_brushed` (256×512) dresses the world
panel + the two portrait windows, `elev_panel_wide` (512×384) the wide
settings window.

The same script also draws the **inventory icon** (`PanelIcon()` →
`UIAtlases/ItemIconAtlas/elevatorControlPanel.png`, 160×160 — the size every
vanilla `Data/ItemIcons` PNG uses), reusing the plate's brushing and the
button shading so icon and block read as one object. Note the icon folder is
`UIAtlases/ItemIconAtlas/`, **not** a top-level `ItemIcons/` — the `/ItemIcons`
string in `Assembly-CSharp` is the vanilla `Data/ItemIcons` path, not a mod
convention. See [[Mod Structure]] / [[Items]]. Drawing a block's icon
procedurally means a code-built model needs no Unity anywhere in the pipeline.

Deploy-kind trap: a PNG under `UIAtlases/**` is normally **xui** (hot-
reloadable). But the *world* panel doesn't read the atlas — `EnsureArt`
`LoadImage`s the PNG off disk once and caches the material, so an
`xui reload` won't refresh it. A world-panel texture change only shows after a
full **restart** deploy (world reload re-runs `EnsureArt`). Touch the C# too
(e.g. `PlateTint`) and it's a restart deploy anyway.

## The back of the panel shows the art mirrored — make the shell opaque

Queue-layered `Sprites/Default` art has no notion of "behind the plate":
the shader is **Cull Off + ZWrite Off**, so mesh winding doesn't hide
anything and a backing quad drawn at a lower queue can't occlude the
higher-queue layers. Viewed from the panel's rear, the buttons, labels and
LED digits paint straight over the back plate **in mirror image** (TextMesh
included — its single-sided geometry doesn't help because the shader
doesn't cull).

Fix: draw the rear shell (back plate + the four edge quads) with an
**opaque, depth-writing material at geometry queue**. All the
`Sprites/Default` art layers depth-test (ZTest LEqual), so once the shell
writes depth, everything behind it clips — the back reads as solid sheet
metal. Face-side layering is unaffected: the art sits at a higher z than
the whole shell, so from the front the depth test always passes.

Shader choice: `Unlit/Texture` (opaque, textured, unlit) when the build
ships it; fall back to `Hidden/Internal-Colored` (always present) with
`_ZWrite=1`, `_SrcBlend=One`, `_DstBlend=Zero` for a flat-colour solid.

**Unlit surfaces must be authored DARK.** `Unlit/Texture` has no colour
property, and an unlit mid-gray (~0.45) texture renders as a glowing white
slab next to lit geometry — unlit pixels don't darken with the scene. Bake
the face art's tint into a copy of the texture (`GetPixels32`, multiply,
`SetPixels32` — textures from `LoadImage` stay readable) so the shell lands
on the same effective colour as the tinted face art (multiply by `PlateTint`,
then use that same average for the `Hidden/Internal-Colored` fallback's
`_Color`). Note the ceiling: `Sprites/Default` is unlit and clamps
`color × texture`, so the sprite's own brightness caps the plate — you can't
brighten past the PNG with tint, only tint or darken it. Elevator's current
plate is bright satin aluminium authored straight into the PNG (~0.75
luminance), so `PlateTint` is fully **neutral** (1,1,1) and the fallback
`_Color` ≈0.73; the plate shows the sprite as-is. (History, same principle:
earlier iterations used a darker sprite lifted by a near-white `PlateTint` —
~0.45 brushed steel, before that ~0.3 bronze.)
Reference: Elevator `src/ElevatorPanelView.cs` (`MakeShellMat`, `TintTexture`).
