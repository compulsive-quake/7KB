# Secondary Camera Feeds (RenderTexture + IMGUI)

Rendering an extra in-world camera (drone feed, security monitor, picture-in-
picture) and putting it on screen in 7DTD. Pattern proven in the `FPV` mod
(`FPVDroneManager.CreateDroneCamera` / `CreatePilotCamera`).

## The pattern

1. `new RenderTexture(w, h, 24, RenderTextureFormat.Default)` + `.Create()`;
   a `GameObject` with a `Camera` whose `targetTexture` is that RT.
2. Copy render setup from `Camera.main`: `cullingMask`, `clearFlags`,
   `backgroundColor`, `allowHDR`, `allowMSAA` — a default-constructed camera
   does not match the game's.
3. Draw it from `OnGUI` with `GUI.DrawTexture(rect, rt, ScaleMode.StretchToFill, false)`.
   IMGUI draws over the game view *and* the (hidden) vanilla UI.
4. Teardown: `Destroy(camera.gameObject)`, `rt.Release()` + `Destroy(rt)`.

## Gotcha: clone the main camera's PostProcessLayer or the feed is dark/flat

The scene renders in **linear color space** and the main camera's
PostProcessing-Stack-v2 `PostProcessLayer` (tonemapping, grading, auto-exposure)
maps that to the final image. A bare secondary camera skips it → noticeably
darker, flatter feed. Fix: add a `PostProcessLayer` to your camera and `Init()`
it with the main layer's `PostProcessResources`, which is only held in the
private field `m_Resources` (reflection). Copy `volumeLayer`,
`antialiasingMode`, `stopNaNPropagation`; set `volumeTrigger` to your camera's
transform so local post volumes apply at *its* position. (In `FPV`:
`ReplicateMainCameraPostFx(main, target)` in `FPVDroneManager.cs`.)

## PiP of the "real" view while a feed owns the screen

While a full-screen feed covers the display, the game's **main camera keeps
rendering** the real world underneath (in `FPV` the player is flipped to
third person, so it's an over-the-shoulder view of the operator's body). You
can't cheaply grab that backbuffer from IMGUI, and leaving a "hole" in the
overlay only shows a 1:1 corner *crop*, not a scaled-down frame. Correct
approach: a second low-res camera (quarter screen res is plenty) that mirrors
`Camera.main` each **LateUpdate** — `SetPositionAndRotation(main.transform...)`
plus `fieldOfView` — into its own RT, drawn as a small rect in the HUD.
`Camera.main` still resolves to the player camera (feed cameras are untagged),
and the RT at screen/4 keeps the screen's aspect so the PiP rect should too.
(In `FPV`: `UpdatePilotCamera` / `FPVHudRenderer.DrawPilotPip`, toggled by
`FPVHudSettings.PilotView`.)

## Layer trick: hide an object from one camera only

Put the object on a spare layer (FPV uses 31) via `SetLayerRecursive` and mask
it out of that camera's `cullingMask` (`mask &= ~(1 << layer)`). Other cameras
keep the full main-camera mask and still see it — how the drone body is hidden
from its own onboard camera yet visible in the pilot PiP and to the player.

Related: [[Remote Camera World Loading (Drone FPV)]],
[[Voxel Light & Scene Brightness]] (why night-vision boosts were abandoned in
favor of post-fx cloning), [[IMGUI Tracker Stack]], [[HUD Safe Zones]].
