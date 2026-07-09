# SelectionBox Overlays (SelectionBoxManager)

`SelectionBoxManager` is the game's system for drawing world-space wireframe/filled
boxes (prefab editor volumes, land claim previews, sleeper volumes). Mods can use it
for their own overlays: `SelectionBoxManager.Instance.CreateCategory(...)` then
`category.AddBox(name, pos, size)`.

## Gotcha: categories start INACTIVE — activate before AddBox

`CreateCategory` calls `SetVisible(false)` on the new category before returning, and
`SelectionCategory.SetVisible` is `gameObject.SetActive` on the category's GameObject.
`AddBox` parents each new `SelectionBox` GameObject under the category transform.

Consequence: a box added while its category is hidden sits under an inactive parent,
so Unity never runs the box's `Awake()`. `Awake()` is what creates the mesh child and
assigns `m_MeshRenderer` / materials. Calling `ShowThroughWalls(bool)` (and anything
else touching `m_MeshRenderer.materials`) on such a box throws
`NullReferenceException` inside `SelectionBox`.

**Fix / rule:** call `category.SetVisible(true)` **before** `AddBox` (or before the
first method that touches the box's renderer), not after configuring the boxes.

```csharp
var cat = SelectionBoxManager.Instance.CreateCategory("MyCat", col, col, col, false, "", 0);
cat.SetVisible(true);                       // MUST come first
var box = cat.AddBox("box1", pos, size);    // Awake() runs here now
box.SetVisible(true);
box.ShowThroughWalls(true);                 // safe
```

Symptom in the log: `NullReferenceException` with top frame
`SelectionBox.ShowThroughWalls (System.Boolean _bShow)`.

## Other notes

- Categories/boxes live on scene GameObjects: a world reload creates a fresh
  `SelectionBoxManager.Instance`, invalidating cached `SelectionCategory` refs.
  Compare the cached manager instance and re-create categories when it changes.
- `ShowThroughWalls` sets the `_ZTest` shader property (8 = Always, 4 = LEqual) on the
  frame and face materials.
- Reuse boxes via `category.TryGetBox(name, out box)` + `box.SetPositionAndSize(...)`;
  `AddBox` with an existing name removes and recreates the box.
- Decompile game classes with `ilspycmd -t <TypeName> "<GameDir>\7DaysToDie_Data\Managed\Assembly-CSharp.dll"`
  (installed as a dotnet global tool).
