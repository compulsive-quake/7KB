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

## Fix — a LateUpdate watchdog, NOT a fixed-frame coroutine (in the switching mod)

A fixed-count re-assert (`for i in 0..3: yield return null; SetFirstPersonView…`)
in a **coroutine is not enough** and was the bug behind "after flying a drone I
can't place another until I crouch". Two reasons it silently fails:

1. **Wrong phase.** Coroutines resume *before* `LateUpdate`. The leftover third-
   person camera controller re-parks the camera in *its own* `LateUpdate`, i.e.
   *after* your re-assert ran — so the frame renders parked anyway.
2. **Too few frames.** `SetControllable(true)` propagates through the camera rig
   over an unknown number of frames; until it does, `SetFirstPersonView(true)`
   won't take. A fixed 3-frame budget can expire before that, then stop trying.

Do it in **`LateUpdate`** on a persistent host (the mod's singleton), and keep
re-asserting **until verified**, not for a fixed count:

```csharp
void LateUpdate() { if (_restoreOwner != null) RunWatchdog(); }

void RunWatchdog() {
    if (Time.time > _deadline) { _restoreOwner = null; return; }   // ~2s cap
    var o = _restoreOwner;
    if (!o.bFirstPersonView) o.SetFirstPersonView(true, false);
    bool parked = ((o.cameraTransform.position + Origin.position)
                   - o.getHeadPosition()).sqrMagnitude > 1f;       // gap > ~1m
    if (parked) { o.SetCameraAttachedToPlayer(true, false); _confirm = 0; }  // wins the frame
    else if (++_confirm >= 4) _restoreOwner = null;                // held FP → done
    if (_rebuildHeld) { _rebuildHeld = false; o.inventory?.ForceHoldingItemUpdate(); }
}
```

- Verify by the **camera→head gap**, not a frame count: only stop once the camera
  has sat on the head for several consecutive `LateUpdate`s (SetControllable has
  propagated and the re-park has stopped). Healthy FP keeps the gap well under 1m.
- `SetCameraAttachedToPlayer(true,false)` resets the camera local pose
  **synchronously**, so doing it in `LateUpdate` is what the frame renders with —
  it beats a controller that re-parked earlier the same frame.
- **Cancel the watchdog when you switch back to third person** for the next drone,
  or it fights your own view switch.
- **`ForceHoldingItemUpdate()`**, not `ShowHeldItem()`: the latter early-outs when
  the held item is unchanged, so it can't un-park a stale transform; the former
  destroys+recreates the model (`m_LastDrawnHoldingItemIndex = -1`). Once per
  restore, to avoid holster flicker.

Implemented in FPV `FPVDroneManager.LateUpdate` / `ArmCameraRestoreWatchdog` /
`RunCameraRestoreWatchdog` (replaced the old `FinalizeFirstPersonRestore`
coroutine). See also [[Player Feedback Sounds]].

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

## The *other* half of the freeze: a stuck input action set (`PlayerActionsLocal.Enabled`)

The camera-park fixes above (watchdog + horizontal reach cap + head-basis
fallback) still did **not** fully cure "after flying a drone I can't place another
until I crouch." A read-only idle diagnostic (fires when the player's aim is stuck
with no menu open) pinned the real stuck field:

```
IDLE AIM STUCK (no menu open): bCanControlOverride=True inputEnabled=False
  inputLocked=False fpv=False camGap=2.03m held=fpvTinyDrone
```

`inputEnabled` here is `PlayerMoveController.playerInput.Enabled`
(`PlatformManager.NativePlatform.Input.PrimaryPlayer.Enabled`). Player aim /
mouse-look only run while that set is enabled (see `PlayerMoveController.Update`:
`flag = !cursorWin && !modal && (playerInput.Enabled || VehicleActions.Enabled)`),
so with it false the crosshair and `HitInfo` freeze and no placement spot resolves.

**`SetControllable` does NOT touch this.** `SetControllable(b)` →
`PlayerMoveController.SetControllableOverride` only sets `bCanControlOverride`.
Restoring control therefore leaves a disabled input set disabled — which is why
the camera watchdog alone didn't fix it. (Likely the disabled input is also *why*
the camera stayed parked — the FP camera rig doesn't reattach cleanly while input
is off — so the two symptoms share this root.)

**Who owns `PlayerActionsLocal.Enabled`: the `ActionSetManager` stack, nothing
else.** `LocalPlayerUI.ActionSetManager` is a stack of `PlayerActionSet`s; only
the **top is Enabled**. Opening a `GUIWindow` that `HasActionSet()`
(`GUIWindowManager.EnableWindowActionSet` → `ActionSetManager.Push`) disables the
player's set and enables the window's; closing it (`DisableWindowActionSet` →
`Pop`) re-enables the one below. A view/HUD teardown can leave a **window action
set orphaned** on the stack (pushed, never popped), so the player's input stays
disabled with **no window on screen** to explain it. Note the game's own
`SetHUDEnabled(FullHide)` just `SetActive(false)`s the XUi GameObject — it does
**not** pop action sets, so hiding the HUD around a feed doesn't cause this by
itself; the orphan comes from a window (e.g. the pre-flight radial, which opens
**modal** — `windowManager.Open(windowGroup, _bModal:true)`).

**Fix — rebuild the stack with the game's own `GUIWindowManager.ResetActionSets()`.**
It pops everything, pushes `playerUI.playerInput` as the enabled base, then
re-pushes any window still *showing*. With no window showing, the player input
ends enabled again; an orphan (window not showing) is simply dropped.

Scope the heal tightly so it never yanks input from another mod's legitimately
open, non-modal UI (the same diagnostic flagged a healthy-camera
`held=elevatorControlPanel inputEnabled=False`, i.e. a real in-use control-panel
window — a *false positive* for "stuck"). FPV heals only when: **holding an FPV
launcher** (`holdingItem.Actions[0] is ItemActionLaunchFpvDrone`), **not actively
piloting** (`!IsBusy`), and **no modal/cursor window open and `!IsInputLocked`**.
Because the radial opens *modal*, the `IsModalWindowOpen()` guard already skips it
while it legitimately owns input; only a closed-but-orphaned set (no modal open,
input still disabled) triggers the rebuild. Read `.Enabled` reflectively —
`PlayerActionSet` lives in the InControl assembly the mod doesn't reference.

Implemented in FPV `FPVDroneManager.LateUpdate` → `HealStuckPlacementInput` /
`IsHoldingLaunchAction` (throttled 0.25s; backs off to 3s if a rebuild didn't
take). Replaced the read-only `DiagnoseIdleControl` probe that identified the field.

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

Fix has **two halves**, both in FPV `FPVDroneManager.TryComputeGroundSpot`:

1. **Reach cap on the horizontal ground distance** (`dx²+dz²` from
   `owner.position`), not the slant. Cast the look ray *farther* than the cap
   (`PlacementRayLength` ≈ 14m vs `MaxPlacementReach` ≈ 9m) so a near floor is
   always *found* regardless of camera height, then reject any resolved spot past
   the horizontal cap. A floor a fixed distance ahead is then placeable at any
   stance; a true horizon/sky gaze still hits nothing.
2. **Parked-camera raycast fallback.** The horizontal cap alone still fails if the
   camera is parked *behind/above* the head (post-recall), because the ray then
   originates from the wrong place — the spot lands far out or behind the player.
   So detect the parked pose by the camera→head gap (`(cam.pos+Origin) − getHead`
   > ~1m) and, when parked, cast from `getHeadPosition()` + `GetLookVector()`
   instead of the camera basis. Use the camera basis in the healthy case for exact
   crosshair alignment (the head bone drifts off the reticle at steep pitch — see
   [[Entities]] "Crosshair alignment") and only drop to the head basis while parked.

With the LateUpdate watchdog above now holding the camera on the head, half (2)
should rarely fire in practice — but it's kept as defence-in-depth (and documents
the consumer-side pattern) in case a frame slips through before the watchdog wins.
