# Camera View Switching (First/Third Person)

How `EntityPlayerLocal.SetFirstPersonView` actually moves the camera, and the
cross-mod gotcha where a temporary first→third→first switch (e.g. FPV's drone
feed) corrupts the aim/laser basis of *other* mods until it's fully restored.

## The key fact: `playerCamera.transform` IS `cameraTransform`

In `EntityPlayerLocal.SetupView`:

```csharp
cameraTransform = cameraContainerTransform.Find("Camera");
playerCamera     = cameraTransform.GetComponent<Camera>();
```

They are the **same transform**. So "read `playerCamera` instead of
`cameraTransform` to dodge a view-switch mod" does *not* work — if one is parked
in the wrong pose, so is the other. There is no separate, always-correct camera
transform to fall back on.

## What SetFirstPersonView does

`SetFirstPersonView(bool fpv, bool lerp)` →

- FP: `SetCameraAttachedToPlayer(true, false)` — reparents `cameraTransform`
  under `cameraContainerTransform`, sets it as first sibling, and resets local
  pose to `Constants.cDefaultCameraPlayerOffset` / identity.
- TP: `SetCameraAttachedToPlayer(false, _)` — `cameraTransform.parent = null`;
  the third-person orbit controller (`vp_FPCamera`) then drives it.

`SetControllable(b)` only flips `bCanControlOverride` (immediate bool), but the
camera controllers re-evaluate it on **their own Update**, not synchronously.
So calling `SetFirstPersonView(true,…)` in the *same frame* you restore control
can fail to take — leaving `cameraTransform` parked in the third-person
(behind/above-the-head) pose for a frame or more.

## The cross-mod symptom

A view-switch mod (FPV drone: first→third while the feed is up→first on
recall/detonate) that doesn't fully restore leaves two transforms stale for a
few frames:

1. **`cameraTransform`** parked behind the head.
2. The **held-item model transform** (`inventory.GetHoldingItemTransform()`)
   stranded on the body / at the lens.

Any mod that builds a world-space aim ray or `LineRenderer` from those (e.g.
**Airstrike** / **RocketTurret** laser designators) then draws from the wrong
basis: the beam balloons into a screen-filling **cone** (near endpoint at/behind
the camera near plane) and the painted target lands on the **operator's own
head** instead of under the crosshair.

## Fix — re-assert first person over a few frames (in the switching mod)

Don't rely on a single inline restore. After handing control back, re-assert for
~3 frames on a persistent (`DontDestroyOnLoad`) MonoBehaviour:

```csharp
for (int i = 0; i < 3; i++)
{
    yield return null;                 // let SetControllable propagate to the rig
    if (owner == null) yield break;
    if (!owner.bFirstPersonView) owner.SetFirstPersonView(true, false);
    owner.SetCameraAttachedToPlayer(true, false);   // force camera back on the rig
    if (i == 0) owner.inventory?.ForceHoldingItemUpdate();  // rebuild held-item xform
}
```

- `SetCameraAttachedToPlayer(true,false)` is public, idempotent, and unconditional
  — it reparents + resets the camera local pose regardless of control state.
- **`ForceHoldingItemUpdate()`**, not `ShowHeldItem()`: `ShowHeldItem`→
  `updateHoldingItem` **early-outs** when the held item is unchanged, so it can't
  un-park a stale transform. `ForceHoldingItemUpdate` destroys+recreates the
  model (`m_LastDrawnHoldingItemIndex = -1`), forcing a clean rebuild. Call it
  once (per restore) to avoid holster flicker.
- `yield` can't sit inside a `try/catch`; wrap the per-frame body in a static
  helper that try/catches internally.

Implemented in FPV `FPVDroneManager.RestoreHudAndPlayerControl` /
`FinalizeFirstPersonRestore`. See also [[Player Feedback Sounds]].
