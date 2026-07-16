# Selection Boxes (SelectionBoxManager)

Part of the [7DTD Modding Knowledgebase](README.md). Drawing colored world-space
boxes from mod code using the game's own selection-box system (the one the prefab
editor, sleeper volumes and trader teleport volumes use). API reflection-verified
against 3.0.x Assembly-CSharp (Elevator mod, 2026-07). In-game rendering confirmed
2026-07: boxes do render on the default camera in normal gameplay, but the solid
translucent faces are sized for editor use — up close (e.g. standing at a
player-scale box) they flood the view with the face color and hide everything
inside/behind the box. For player-facing previews prefer a custom dashed
wireframe (below); the Elevator mod dropped SelectionBoxManager entirely for it.

## API surface

```csharp
SelectionBoxManager mgr = SelectionBoxManager.Instance;   // static field; scene object — null-check it

// Category = a named group of boxes with shared colors
SelectionCategory cat = mgr.CreateCategory(
    "MyCategory",
    colSelected, colUnselected, colFaceSelected,   // UnityEngine.Color, alpha ~0.2-0.35 for faces
    false,                                          // bCollider — false for display-only boxes
    "",                                             // collider tag (unused when bCollider false)
    0);                                             // layer — 0 renders on the default camera

SelectionBox box = cat.AddBox("myBox", blockPos, sizeInBlocks, false, false); // Vector3i world block coords
box.SetPositionAndSize(blockPos, size);   // move/resize an existing box
box.SetVisible(true);
box.ShowThroughWalls(true);               // x-ray outline
box.SetFrameColor(Color.red);             // edge lines, separate from face color
box.SetCaption("label text");             // floating TextMesh above the box
box.SetSizeVisibility(true);              // per-axis dimension labels like the prefab editor
cat.TryGetBox("myBox", out box);          // safe lookup — use before AddBox to avoid duplicates
cat.RemoveBox("myBox");
cat.SetVisible(true);
```

## Gotchas

- `SelectionBoxManager.Instance` is a MonoBehaviour in the game scene: it is
  **destroyed and recreated on world reload**. Cache neither categories nor boxes
  across worlds — compare `Instance` to the one you created your category on and
  rebuild when it changes.
- `mgr.GetCategory(name)` is a plain dictionary lookup — prefer tracking your own
  `SelectionCategory` reference rather than probing (may throw on missing key).
- Positions are **world block coordinates** (`Vector3i`); the system handles the
  floating origin internally (`SelectionBox` re-anchors on origin shift).
- `SelectionCategory` face colors come from the category's active/inactive colors;
  per-box `SetAllFacesColor` and `SetFrameColor` override them.
- `SetCaption` / `SetSizeVisibility` text is a world-space `TextMesh` using the
  GUI text shader: **huge** (sized for the prefab editor's zoomed-out camera),
  drawn through all geometry (`ZTest Always`), and mirrored when viewed from
  behind (`Cull Off`). Up close it reads as unusable green garbage. There is no
  API to shrink it — if you need readable in-world labels, skip captions and
  render your own text (below).

## Rolling your own: dashed wireframe + fitted per-face labels

When SelectionBox's solid faces / giant captions don't fit (verified in-game,
Elevator mod 2026-07, `src/ElevatorBoundsRenderer.cs`):

- **Dashed box outline**: one `Mesh` with `MeshTopology.Lines` (pairs of verts
  per dash), material from `Hidden/Internal-Colored` with `_ZTest Always`,
  `_ZWrite 0`, SrcAlpha/OneMinusSrcAlpha, `renderQueue 4000` (see
  [[Runtime Procedural VFX]] for the material recipe). No GL/OnRenderObject
  needed; rebuild only when the box changes.
- **Many boxes**: batch them all into a single mesh (Elevator's
  `ElevatorFloorBoxRenderer` draws every floor box in one rebuild) and set
  `mesh.indexFormat = IndexFormat.UInt32` — lots of tall dashed boxes can pass
  the 64k default vertex cap. Tint via per-vertex colors so one material serves
  differently-colored renderers.
- **Floating origin**: write mesh verts in absolute world coords and set
  `transform.position = -Origin.position` every `LateUpdate`.
- **Built-in font on Unity 2022 (7DTD 3.0)**: `Resources.GetBuiltinResource
  <Font>("LegacyRuntime.ttf")` — `"Arial.ttf"` no longer exists (throws). Try
  both for forward-compat. `TextMesh` lives in `UnityEngine.TextRenderingModule`
  (add the csproj reference).
- **Fit text inside a face**: set `TextMesh.fontSize` high (64) for atlas
  crispness, then measure `renderer.bounds` at `localScale = 1` and scale by
  `min(faceW*margin/w, faceH*margin/h, absoluteCap/h)`. Works because the faces
  are axis-aligned: the world AABB cleanly splits width (x or z) vs height (y).
- **No mirrored text from behind**: the font material draws through walls and
  is double-sided, so per frame enable only labels whose face normal points at
  the camera: `Dot(normal, camPos - labelPos) > 0` with
  `EntityPlayerLocal.cameraTransform` (reflection-verified field). Standing
  *inside* the box hides all labels — acceptable, beats mirrored garble.

## Billboarded labels with an outline (readable over bright boxes)

For labels floating inside/over colored boxes (Elevator mod 2026-07,
`src/ElevatorFloorLabels.cs`), face-fitted text isn't enough — white text on a
bright green face needs an outline, and shaft-style views (looking straight
down) need billboarding instead of per-face culling:

- **Screen-aligned billboard**: set the label root's `rotation = cameraTransform.rotation`
  every `LateUpdate`. Readable from any angle, never mirrored, never degenerate
  (unlike `LookRotation` with world-up when looking straight down).
- **Outline**: `TextMesh` has no outline support. Fake it with 8 black copies of
  the TextMesh offset in a ring (4 cardinal at ±d, 4 diagonal at ±0.7d) behind a
  white main copy. d ≈ 5% of the line height reads well (line height at
  `fontSize 64` / `characterSize 0.1` is ~0.64 local units → d = 0.03).
- **Draw order**: clone `font.material` twice and set explicit `renderQueue`s
  (e.g. outline 4001, text 4002 — above SelectionBox faces at ~3000 and any
  custom lines at 4000). Same-queue transparent sorting by distance is a tie at
  identical positions; explicit queues make outline-under-text deterministic and
  keep labels on top of the box faces.
- **Cloned font materials go stale**: dynamic fonts regrow their glyph atlas and
  may swap in a new texture; clones keep pointing at the old one. Subscribe to
  `Font.textureRebuilt` and re-copy `font.material.mainTexture` into the clones.
- Sizing/fitting and the floating-origin trick are the same as the per-face
  recipe above (measure `renderer.bounds` at scale 1, then uniform-scale to
  `min(fitWidth*margin/w, maxHeight/h)`).
