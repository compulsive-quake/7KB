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

**Dressing the placement ghost with sample content.** The flip side of that
split is that the held-item preview is the *bare* chassis — a blank plate with
a dead screen, which tells the player nothing about where the buttons land.
Give the preview (and only the preview) a stand-in face:

- Hook `ItemClassBlock.CreateMesh` with a **Postfix** and gate on
  `_purpose == BlockShape.MeshPurpose.Preview`. That's the single funnel every
  held-block ghost goes through, and the purpose flag is what separates it from
  the chunk and mover-proxy paths (which get their real dressing elsewhere).
  One overload, params `_blockValue` / `_purpose` / `__result` — 3.0.x.
- `__result` is the bare **wrapper**; anchor on `__result.GetChild(0)` (the
  wrapper trap below).
- **Build the sample face ONCE into a template and `Instantiate` it per
  preview** — do *not* rebuild it per call. `RenderDisplacedCube` rebuilds the
  ghost on every focus-block/rotation/face change and tears the old one down
  with a plain `Object.Destroy`; its mesh recycling (`VoxelMesh.AddPooledMesh`)
  is explicitly skipped for `BlockShapeModelEntity`, and destroying a
  GameObject never frees the `Mesh` assets under it. Rebuilding per preview
  leaks a mesh per quad every time the crosshair moves to a new block.
  `Instantiate` copies `MeshFilter.sharedMesh` **by reference**, so a template
  shares its meshes with every clone. Park it under an inactive holder (same
  active-prefab-under-inactive-holder trick as the model itself).
- Reuse the *real* rebuild path with a synthetic data object (Elevator feeds it
  a six-floor elevator parked at floor 1) so the preview cannot drift from what
  actually gets placed.
- No teardown needed on your side: `RenderDisplacedCube` deep-copies materials
  for the preview (`Renderer.materials`, per renderer) and destroys those copies
  in `DestroyPreview`, so your shared art set is never touched; and its
  `disableAllComponents` / emission / `SetLayerRecursively(…, 2)` passes run
  *after* your postfix, so the added art is treated exactly like the chassis.
  Note `TextMesh` is a `Component`, not a `MonoBehaviour`, so
  `disableAllComponents` leaves labels alone.

Reference: `ElevatorPanelView.DressPreview` / `PreviewFace` +
`Patch_ItemClassBlock_CreateMesh`.

**More than one code-built model in the same mod.** Elevator ships two now —
`elevatorPanelModel` (the cab panel) and `elevatorCallButtonModel` (the hall
call button, `ElevatorCallButtonModel.cs` + `ElevatorCallLights.cs`). Two
things generalise:

- **One shared art set, two chassis.** Both models build from the same
  `ElevatorPanelArt` materials/meshes (`EnsureArt` latches, so whichever block
  inits first builds the set; the second reuses it). Keeping all runtime
  materials owned by one module is what makes the `DestroyObject` exemption
  below sufficient.
- **Each model needs its OWN `getPrefab` and `DestroyObject` prefix**, keyed on
  its own `modelName`. Two prefixes on the *same* method is fine — Harmony runs
  all of them, each acts only on its model, and if any returns false the
  original is skipped (so the non-matching prefix returning true is a harmless
  no-op). Don't try to fold both models into one prefix unless they truly want
  identical handling.

**The one-block Z shift is a property of the whole `elevatorDoor` panel
family, not just the inside panel.** A code-built model authored with its
geometry at anchor-space z≈0 (the natural way) mounts **one block deeper** than
the vanilla prefab it copies, so its `ModelOffset` Z must be **1.0 less** than
vanilla's. Confirmed for `elevatorInsidePanel` (vanilla base ladder z
0.5/1/1.5 → mod −0.5/0/0.5) and applied to `elevatorOutsideButtonPanel`
(vanilla ladder −0.5/−1/−1.4 → mod −1.5/−2/−2.4). If a code-built pad lands a
block off the aimed block, this Z term is the first knob. Note the button's
`ModelOffset` X is **0** (the pad is x-centred, footprint x ±0.079) — only the
*inside* panel carries the −0.25 X term (its plate is centred x +0.242).

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

## A rig only exists where your mod data does

The rig loop for this pattern iterates your own per-block dictionary, not the
world's blocks: `foreach (kv in MyMod.Signs) { get BlockEntityData at kv.Key;
build/refresh rig }`. That is what keeps it cheap — but it means a block the
player has placed and **never configured** has no entry, so it gets no rig, so
it renders as the bare chassis. Any "not set up yet, press E" message you draw
in the view is unreachable for exactly the blocks that need it.

Fix it at placement: override `OnBlockAdded` and create the entry (with a
sensible default link — Elevator's floor directory picks the nearest elevator,
so a placed sign is already listing the right shaft).

```csharp
public override void OnBlockAdded(WorldBase _world, Chunk _chunk, Vector3i _pos,
    BlockValue _bv, PlatformUserIdentifierAbs _addedByPlayer)
{
    base.OnBlockAdded(_world, _chunk, _pos, _bv, _addedByPlayer);
    if (_bv.ischild || MyMover.IsCarrying(_pos)) return;   // see below
    MyMod.GetOrCreate(_pos);
}
```

Ground truth on when it fires (`Chunk.SetBlock`, 3.0.x): **on placement and on
any other real block change — NOT on chunk load.** `Chunk.SetBlock` calls
`OnBlockAdded` only under `_notifyAddChange` and only when `blockValue.type !=
_blockValue.type`; a same-type write goes to `OnBlockValueChanged` instead, and
loading a chunk from disk doesn't run this path at all. So it is the exact
mirror of `OnBlockRemoved`, and the two share a caveat in a mod that moves
blocks: a mover that lifts and re-places blocks trips *both*, so guard both
with the same "is my mover carrying this position" test or a riding block
loses (or re-defaults) its config on landing.

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

**Make the rim colour a parameter, not a constant.** The rim's job is to hold
the glyph off *the surface it sits on*, so it takes the SURFACE's value, not a
fixed dark. A hardcoded dark rim is right for light-on-dark engraving and
actively wrong on a light applied plate — there it just fattens dark lettering
into a blob. Elevator's `ElevatorPanelArt.MakeText` has both: the default
`OutlineColor` (near-black) for the cab panel's light text on dark metal, and
an explicit light `CellTextOutline` for the floor directory's dark text on its
bright name plates.

**One texture, two tints — author it near-white.** `Sprites/Default`
multiplies colour × texture, so a tint can only ever *darken*. A strip/plate
texture meant to serve both a dark and a light variant (Elevator's
`elev_directory_cell.png`: the applied row plate behind both the charcoal
number cell and the bright name cell) must therefore top out near 255 and
carry only *shape* — bevels, a dome, edge falloff. Anything mid-grey in the
texture caps the light variant.

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
full **restart** deploy (world reload re-runs `EnsureArt`). Same for the
world plate under `resources/`, which isn't an atlas sprite at all.

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
brighten past the PNG with tint, only tint or darken it.
Reference: Elevator `src/ElevatorPanelArt.cs` (`MakeShellMat`, `TintTexture`).

## Making a world-space surface read as METAL

Long-running failure on Elevator's panel: every iteration of the plate
tint (~0.3 bronze → ~0.45 brushed steel → neutral 1,1,1 on a bright sprite →
0.46 back down again) produced the same complaint — "it looks like a white
screen, not aluminium". Tint was never the variable. Four things were.

**1. An unlit quad cannot be metal.** Metal reads as metal because of a
specular highlight that *moves with the viewer* plus an environment
reflection. `Sprites/Default` has neither: same flat value from every angle
in every light — a printed poster. No amount of texture work fixes this on
its own, and no tint ever will.

**2. Broad shading is the cue — grain is not.** The old plate spanned
~158..212 of 255 (a ~9% value spread) then got multiplied flat by the tint.
That is a grey card by construction. Bake in:
- a **reflection ramp** — bright ceiling at the top, dark floor at the
  bottom, soft horizon slightly above centre, plus a soft specular band
  across the upper third. Cheapest "this is reflective" signal there is, and
  on a stock-Unity-shader quad (which gets no voxel light) it is the *only*
  thing keeping the plate from going flat;
- a broad **cylindrical roll** darkening toward the left and right edges, so
  the sheet isn't dead flat.

**Do not paint brush grain.** This note previously recommended "two-octave
scratches" and an anisotropic specular band. Measuring vanilla's own art
killed both: at any distance a player actually views these panels from, fine
per-column noise never resolves as a satin brush — it resolves as **full-
height stripes**, and the report you get back is "cartoonish vertical lines".

**Ground truth — vanilla's panel metal** (`elevatorDoor_d` / `_n` / `_rmom`
in `Data/Addressables/Standalone/automatic_assets_entities/doors.bundle`, the
albedo set shared by `elevatorInsidePanel` and `elevatorOutsideButtonPanel`):

| map | plate field |
|---|---|
| albedo `_d` | **RGB(156, 147, 134)**, σ ≈ 1.0 over the whole field |
| normal `_n` | flat (127, 126) everywhere — no grain in the surface either |
| `_rmom` | roughness ≈ 0.22, metallic ≈ 0.83 |

So vanilla's aluminium is a **dead-flat warm greige** with a **flat normal**,
and every metal cue — the sheen sliding across the call button, the warm cast
under the sun — comes from the shader, not from anything painted on. Match
that hue with a luminance-preserving multiplier over a plain grey ramp
(156:147:134 normalised through 0.299/0.587/0.114 ⇒ **×1.053 / ×0.992 /
×0.905**), and add nothing but sub-1-LSB dither to stop the ramp banding.
Elevator's `MetalPlate`/`CabPlate` in `tools/gen_ui_sprites.ps1` do exactly
this. Use the same multiplier on the **button sprites' rims**, not just the
plate: vanilla's discs are brass-rimmed against a near-neutral sheet, and that
warm-on-cold contrast is most of what reads as "elevator" at a glance — a
cool-rimmed disc on a dark plate reads as a chrome pip. Watch the tints on
*neighbouring* materials too — a cool blue-grey
shell multiplier (Elevator had `0.88,0.88,0.91`) pulls the folded edges back
toward steel while the face goes warm, and the panel comes apart.

Sampling recipe (UnityPy 1.25 is installed; ~10 s):

```python
import UnityPy, statistics
env = UnityPy.load(r"…\automatic_assets_entities\doors.bundle")
for o in env.objects:
    if o.type.name == "Texture2D":
        d = o.read()
        if d.m_Name == "elevatorDoor_d":
            px = list(d.image.convert("RGB").crop((600,300,1400,1500)).getdata())
            print([round(statistics.mean(p[c] for p in px)) for c in range(3)])
```

Don't commit the dumped PNGs — they're shipped game art, and the numbers are
what you needed.

**Do not bake a bevel.** It's the obvious next idea and it's wrong: any ring
of contrast at a fixed distance from the edge reads as a *border painted on
the panel*, whatever its colour. Elevator shipped a dark seam (read as a
black border), then a bright chamfer (read as a white one), before dropping
edge-aligned detail entirely. Let the plate run clean to the boundary — the
silhouette comes from the shell's edge quads catching the light differently
to the face, which is how a real folded sheet shows its edge anyway.

**3. Texture aspect must match the quad's aspect.** UVs are the full 0..1
rect, so a 1:2 PNG on a 1:4.15 face stretches 2.07× and smears the grain out
of existence. Match it: Elevator's 0.483 × 2.003 m face gets a 256×1024 PNG
(~520 px/m, in line with vanilla block texel density). Also set
`filterMode = Trilinear` and `anisoLevel = 8` — a 2 m plate is normally seen
at a grazing angle, which is exactly the case aniso exists for.

**4. A world texture and a XUi sprite cannot be the same file.** 2-D UI is
composited unlit, so XUi plates want bright and flat; a world plate must
carry its own lighting, so it wants dark and high-contrast. Sharing one PNG
means one of the two is always wrong — that fight *is* the whole tint
history above. Split them: XUi sprites stay in `UIAtlases/UIAtlas/`, the
world texture goes in `resources/` (keeping it out of the atlas, which it
was never read from anyway — see the deploy-kind trap above).

**4b. …but a XUi window that depicts the block wants a THIRD plate.** Elevator's
in-cab floor picker (the window the horn pops up while you sit in the car) is
looking at the panel on the wall, so a bright settings-sheet plate reads as a
different object. That window gets its own atlas sprite —
`elev_panel_cab.png`, `CabPlate` in `tools/gen_ui_sprites.ps1` — authored to
the *world* rules (reflection ramp, vanilla's warm-greige hue, no grain) but
lifted and range-compressed, because a UI sprite never gets the scene's
exposure back and the dark button discs still have to show against it. Author
the ramp against the window's own furniture (specular band behind the header,
floor bounce along the footer), and keep the sprite at the window's aspect —
the ramp is placed by proportion, so a stretched sprite puts the highlight in
the wrong place. Same knock-on as the world plate: a dark plate flips every
label in the window from near-black to light, and the *rest* of the mod's
windows can stay bright — one dark window among light ones reads as intent
when it depicts a dark object.

Match the live parts too, or the illusion breaks the moment the car moves: the
window's readout and lamps should be driven off the same helpers/palette the
world panel uses (`ElevatorPanelArt` colours, `ElevatorPanelView.NearestFloor`,
the same blink period), not re-typed constants.

### Using a lit shader (`Standard`) on code-built block art

Worth it — it buys the real moving specular — but three gotchas:

- **Hand-built meshes need `mesh.normals`.** A lit shader on a mesh with no
  normals renders **black**. For a double-sided quad built from one set of 4
  verts (both windings, shared verts) there is only one normal per vertex, so
  use the outward one. With Elevator's UV-ordered in-plane axes that is
  `Vector3.Cross(up, right)`, not `Cross(right, up)`. Set `tangents` too.
- **7DTD's voxel lighting does not reach a stock Unity shader** (it rides in
  the game's own block shaders). In an unlit shaft a `Standard` plate goes
  black — worse than the white it replaced. Fix with an **emission floor**:
  `_EmissionMap` = the plate texture, `_EmissionColor` = a grey,
  `globalIlluminationFlags = EmissiveIsBlack` so it stays a pixel-shader
  effect. Reads as the backlit face a real cab has.

  **Keep that floor LOW — ~0.10-0.13, not ~0.30.** This is the single
  highest-leverage number on the surface and it is easy to get backwards.
  Because voxel light never arrives, emission is not a floor under the
  lighting, it very nearly *is* the lighting: at 0.30 it accounted for ~90% of
  Elevator's rendered panel and the `Standard` setup underneath was decoration.
  Symptom is a plate that looks identical in a lit lobby and a black shaft, and
  reads far brighter than vanilla's next to it. See "Measure the render, not
  the texture" below for how to pick the number instead of guessing it.
- **`StripLighting` must not touch the lit renderers.** `reflectionProbeUsage
  = Off` leaves a metallic material with nothing to reflect and it collapses
  straight back to flat grey. Set `lightProbeUsage`/`reflectionProbeUsage` to
  `BlendProbes` (falls back to the sky probe when no local probe exists).
  Shadow *casting* stays off — an inset plate only shadow-acnes itself.
- **Light the WHOLE chassis or none of it.** A lit face with an unlit shell
  (back + edge quads) looks fine in the sun and falls apart indoors: the face
  dims with the scene, the unlit edges don't, and the panel reads as a dark
  plate with glowing trim. Put the shell on the same shader, a little rougher
  and darker — it still writes depth, so the occlusion above is unaffected.

Tuning: **move the ALBEDO with the shader, or you will chase your tail.**
Elevator ran metallic/smoothness at 0.9/0.74 ("polished chrome under the
midday sun"), reacted by halving to 0.45/0.38, and this note used to say that
landed it. It didn't — it traded chrome for a flat card whose look came
entirely from a steep painted gradient and a fat emission floor. The step that
was missing: **a metal's reflection is multiplied by its albedo**, so high
metallic only means chrome when the albedo is bright. Darken the plate texture
(Elevator went 76..184 → 62..110) and vanilla's own measured `_rmom` numbers —
metallic 0.83, smoothness 0.78 (roughness 0.22) — land on vanilla's own look
rather than a mirror. Prefer copying the game's measured values over inventing
a pair; you already dumped the map to get the albedo hue.

Whatever you change, pull the texture's baked specular band by the same factor
— a baked highlight and a real one at different strengths visibly fight. And
once the material reflects for real, the baked ramp should get *flatter*, not
just dimmer: it was standing in for a reflection you now have.

### Measure the render, not the texture

Eyeballing "does this match vanilla" across a screenshot fails, because the
texture value, the emission floor and the scene all compound. Two cheap
measurements settle it in a minute, and both beat a deploy/relaunch cycle:

**1. Sample the screenshot.** Put your block next to the vanilla one it should
match, screenshot, and read actual pixels — `System.Drawing` from PowerShell,
average + min/max luminance over a rect on each plate field. Elevator's
numbers, for calibration: vanilla `elevatorInsidePanel` renders at **luminance
25..33, near-neutral**, in a lit indoor room (all its contrast lives in the
specular highlight and the brass button rims, not the field). The mod's three
plates were at 76, 90 and 108 — i.e. the problem was never subtle, but no
amount of staring had produced a number.

**2. Predict the render from the texture.** While emission dominates, the
rendered value is just `sRGB(linear(texel) × _EmissionColor)`. Run the plate
PNG through that and compare against the sampled vanilla figure *before*
deploying — Elevator's retune predicted 18..37 (avg 25) and that is what
shipped. Also worth compositing a quick GDI+ mock of the whole surface (plate
+ cells + button sprites at their real tints) to check contrast between layers,
since unlit sprite art is a straight multiply and doesn't go through emission
at all. Consequence worth knowing: **once the plate is dark, applied cells must
tint UP, not down.** A cell tinted below the board's rendered value has nothing
for its bevels to show against and the row dissolves into the plate.

### One generator, one metal — don't UV-slice across surfaces

If a mod hangs several plates on the wall (Elevator: cab panel, hall call pad,
floor directory board), author them from **one function at one ramp and one
tint**, emitting one PNG per surface at that surface's aspect. Two failure
modes this avoids:

- Three separately-tuned textures drift into three different alloys. Elevator's
  sign was authored "bright stainless, because a hall sign is a different piece
  of metal to a cab panel" — defensible in isolation, and next to the panel it
  read as printer paper taped to the wall.
- **Sharing one texture and slicing it by UV** fixes the alloy but moves the
  highlight. The call pad took the middle half of the panel's 1:4.15 texture:
  right aspect, but that texture's specular band is placed for the upper third
  of a 2 m panel, so it landed on the pad's top edge. The band's position is
  the one thing that legitimately differs per surface — make it the parameter.

Same rule for the palette: put the accent colours in one shared class and use
them everywhere. Elevator's call arrows were green-up/amber-down while every
other lamp, LED and halo in the mod was amber; the arrow shape already carries
the direction (as it does on every real hall button), so the colour was free to
say "same system" instead.

Trim: once the plate is real metal, **delete the dark border frame**. A
quad-drawn dark edge strip is a UI convention that reads as a black outline
painted on the panel — and don't replace it with a baked bevel either (see
"Do not bake a bevel" above). The shell's edge quads should take a thin UV
sliver off the matching side of the plate so the brushing continues round the
fold instead of restarting.

`Standard` can be stripped from a build, so keep the `Sprites/Default` path
as the fallback; with the texture authored per (2) it still reads as metal,
just without the moving highlight. Log which path was taken.

Knock-on: a dark plate flips the label scheme. Elevator's floor names were
near-black (28,29,33) with a light rim, sized for a near-white plate; on real
metal they invert to light (232,235,240) with a dark rim — which is what a
real engraved-and-filled cab panel looks like anyway. Flip **every** surface at
once, including the ones with a good local reason not to: Elevator's directory
sign kept dark-on-light rows on the (correct) argument that a real lobby board
has a dark number plate on a light backing, and the result was the one surface
in the mod reading the other way round. Polarity is a mod-wide decision — the
applied-plate read survives inversion, since it comes from the strip texture's
bevels and the value *step* against the board, not from which is darker.

Don't forget the **inventory icons** when the world art changes value. They
mirror the same ramp by proportion, so they go stale silently — but they can't
just take the world numbers either: the icon grid composites unlit on a dark
field, where a true-to-world dark plate is a black tile. Stamp the same ramp
and hue with a lift factor (Elevator: ×1.55).

Reference: Elevator `src/ElevatorPanelArt.cs` (`MakePlateMat`,
`EnableLighting`, `LoadTexture`), `tools/gen_ui_sprites.ps1` (`WorldPlate`).
