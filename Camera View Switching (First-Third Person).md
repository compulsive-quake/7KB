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

## Defending the *consuming* mod (when the switcher is vanilla)

The re-assertion fix only helps when you control the switching code. **Vanilla
view switches don't run it** — e.g. taking over a powered SMG/auto turret parks
the camera the same way, and you can't patch its restore. So a designator-style
mod that reads the camera must **detect and self-heal on its own**, not trust the
camera blindly.

Detection without a clean reference transform: compare the camera position to the
head. They're the same space once you shift the render-space camera back to world
space — `cam.transform.position + Origin.position` vs `holder.getHeadPosition()`.
In real first person that gap is well under a metre; a parked third-person camera
is metres behind/above. Past ~1 m, treat the camera as parked.

When parked:
- **Aiming**: fall back to `getHeadPosition()` + `GetLookVector()` for the raycast
  origin/direction. Casting from the parked camera is what drops the strike on the
  operator's own head.
- **Laser origin**: skip the camera/held-item basis (it's behind the lens → cone);
  use the head/look basis instead.
- **Self-heal**: while the tool is out the player is in first-person gameplay, so
  if `bFirstPersonView` is still true but the camera is parked, call
  `SetCameraAttachedToPlayer(true,false)` (resets the camera pose *synchronously*,
  so it's usable the same frame) and `inventory.ForceHoldingItemUpdate()` once
  (throttled) to un-park the held model. Skip the heal when `bFirstPersonView` is
  false so a *deliberate* third-person toggle isn't yanked back.

Implemented in RocketTurret `ItemActionRocketTurretPointer.IsCameraParked` +
`RocketTurretLaserView.LateUpdate` (self-heal) + `TryGetPaintedTarget` (aim
fallback). Note: the earlier "read `playerCamera` instead of `cameraTransform`"
workaround there was a no-op for exactly the reason above — they're one transform.

## Don't measure a crosshair *reach* limit along the camera ray

Same camera-height coupling bites any feature that resolves a crosshair spot and
caps its **reach**. FPV's drone placement raycast from `cameraTransform` capped
reach by the **slant distance along the ray**. That distance is
`cameraHeight / sin(down-pitch)`, so it scales with how high the camera sits:

- **Stance**: the standing eye is ~0.7m above the crouched one, so a nearby floor
  that a crouched player could place silently vanished when standing (longer
  slant blew past the cap) — and reappeared on crouch. Bumping the cap (5m→9m)
  only moves the shallow-angle edge; it doesn't decouple it.
- **Post-recall parked camera** (above): the camera lingering near its
  third-person pose is even higher, so the slant to a near floor exceeds the cap
  and the placement ghost disappears until the camera resettles (or you crouch /
  move, which forces a rig recompute). The classic report: "after flying a drone
  and pressing Esc I can't place the next one until I crouch."

Fix: cap by **horizontal ground distance from the player** (`dx²+dz²` from
`owner.position`), not the slant. Cast the look ray *farther* than the cap
(`PlacementRayLength` ≈ 14m vs `MaxPlacementReach` ≈ 9m) so a near floor is always
*found* regardless of camera height, then reject any resolved spot past the
horizontal cap. A near floor a fixed distance ahead is then placeable at any
stance and survives a not-yet-resettled camera; a true horizon/sky gaze still
hits nothing. Implemented in FPV `FPVDroneManager.TryComputeGroundSpot`.
