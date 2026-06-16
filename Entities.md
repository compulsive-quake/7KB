# Entities

Part of the [7DTD Modding Knowledgebase](README.md). Covers custom entity classes (NPCs, animals, zombies) defined via `Config/entityclasses.xml`.

For drivable vehicles (motorcycles, cars, hover bikes, mounts) see [Vehicles](Vehicles.md) — vehicles are a specialised `EntityVehicle` subtree with their own `vehicles.xml` schema, attach/seat pipeline, and camera handling.

---

## Defining a Custom Entity

Entities extend existing vanilla entity types. All definitions are XPath patches to `entityclasses.xml`:

```xml
<configs>
    <append xpath="/entity_classes">
        <entity_class name="myCustomZombie" extends="zombieMale">
            <property name="Mesh" value="Assets/Prefabs/Entities/ZombieMale.prefab" />
            <property name="SizeExtentsOverride" value="0.5,1,0.5" />
            <property name="SizeExtentsOverrideScale" value="1.2" />
            <property class="AITask">
                <property name="task1" value="BreakBlock" />
                <property name="task2" value="DestroyArea" />
                <property name="task3" value="ApproachAndAttack" />
            </property>
            <drop event="Destroy" name="foodRottingFlesh" count="0,3" prob="0.5" />
            <drop event="Destroy" name="resourceBone" count="0,2" prob="0.5" />
        </entity_class>
    </append>
</configs>
```

---

## Key Properties

| Property | Notes |
|---|---|
| `extends` | Name of the vanilla entity class to inherit from |
| `SizeExtentsOverride` | Collision box size `"x,y,z"` |
| `SizeExtentsOverrideScale` | Scale multiplier applied on top of extents (1.0 = default) |
| `MaxHealth` | Hit points |

Use `extends` to inherit all AI, animations, and properties from a vanilla entity — then only override what you change.

---

## AI Tasks

The `AITask` property class controls what actions the entity takes:

```xml
<property class="AITask">
    <property name="task1" value="BreakBlock" />
    <property name="task2" value="DestroyArea" />
    <property name="task3" value="Territorial" />
    <property name="task4" value="ApproachAndAttack" />
</property>
```

Common task values: `BreakBlock`, `DestroyArea`, `Territorial`, `ApproachAndAttack`, `Wander`, `Investigate`.

---

## Drops

```xml
<drop event="Destroy" name="itemName" count="min,max" prob="0.0-1.0" />
<drop event="Harvest" name="resourceLeather" count="1,3" prob="0.6" />
```

- `event="Destroy"` — drops when entity is killed
- `event="Harvest"` — drops when player harvests the corpse
- `prob` — probability 0.0–1.0

---

## Size Scaling Variants

To create differently-sized variants of the same entity (e.g. progression tiers):

```xml
<entity_class name="myZombieSmall"  extends="myCustomZombie">
    <property name="SizeExtentsOverrideScale" value="0.8" />
</entity_class>
<entity_class name="myZombieLarge"  extends="myCustomZombie">
    <property name="SizeExtentsOverrideScale" value="1.4" />
</entity_class>
```

---

## Visual Customization Properties

Additional properties for controlling entity appearance (used when modifying existing entities):

```xml
<append xpath="/entity_classes/entity_class[@name='zombieMale']">
    <!-- Custom map/compass icon -->
    <property name="entity_icon" value="ui_game_symbol_zicon" />

    <!-- Tint the entity's texture in a custom color (hex RGB, no #) -->
    <!-- param1 = comma-separated Unity material property names to tint -->
    <property name="XTintColor" value="FF00E8"
              param1="_Color,_TintColor,_EmissiveColor" />

    <!-- Replace one of the entity's textures at runtime -->
    <!-- param1 = material path inside a unity3d bundle -->
    <property name="ReplaceTexture" value="2"
              param1="#@modfolder(MyMod):Resources/bundle.unity3d?Materials/MyMaterial.mat" />
</append>
```

---

## Modifying Vanilla Entities

To change a property on an existing entity:

```xml
<configs>
    <set xpath="/entity_classes/entity_class[@name='playerMale']/effect_group/passive_effect[@name='BagSize']/@value">100</set>
</configs>
```

---

## Local Player Camera View (first ↔ third person, C#)

`EntityPlayerLocal` controls which view the local player is in. Useful when a
mod takes over the screen (e.g. a remote camera / drone feed) and needs the
player's body to look right to *other* cameras.

- `bFirstPersonView` (public bool) — current view state; read it to remember
  the player's choice before changing it.
- `SetFirstPersonView(bool firstPerson, bool lerpPosition)` — the canonical
  toggle. It swaps the visible model (`switchModelView` →
  `emodel.SwitchModelAndView`), moves/attaches the camera, sets the `.IsFPV`
  cvar, fires `MinEventTypes.onSelfChangedView`, and toggles
  `characterMatrixOverride`. Pass `lerpPosition: false` for an instant snap.
- `SwitchFirstPersonViewFromInput()` is the input wrapper; it refuses to switch
  when `vp_FPCamera.Locked3rdPerson`, when `AttachedToEntity != null` (in a
  vehicle/turret), or when `CameraRestrictionMode > 0`. Call
  `SetFirstPersonView` directly to bypass those guards.

**Headless first-person body gotcha:** in first person the player renders a
*headless* body model (the head is hidden so it doesn't clip the FP camera).
Any other camera (a second/render-texture camera looking back at the operator)
therefore sees a headless body. Fix: call `SetFirstPersonView(false, false)`
while that camera is active so the full third-person model (with head) renders,
and restore the saved `bFirstPersonView` afterwards. The model sits on layer 24
in *both* views (`SetModelLayer(24)` runs for FirstPerson and ThirdPerson
alike), so a camera that already renders your FP arms will also render the TP
body — no culling-mask change needed; only the visible mesh/head swaps.

`EnumEntityModelView` has just two values: `FirstPerson`, `ThirdPerson`.

Used by the FPV drone mod: when the drone feed activates it flips the operator
to third person so the drone sees a complete body, and restores first person
when control returns.

**Gotcha — restore order vs. `SetControllable`:** the player camera rig is bound
to controllability. If you `SetControllable(false)` while taking over the screen,
you must restore control **before** calling `SetFirstPersonView(true, …)`.
Re-applying the view while the player is still flagged non-controllable can fail
to take, leaving `cameraTransform` parked in its third-person pose. Anything that
reads `cameraTransform` for aiming (the Airstrike / RocketTurret laser pointers
build their beam *origin and direction* from it) then draws from the wrong basis
— the lasers look broken until the player toggles view manually. Order on
teardown: `SetControllable(true)` → restore HUD → `SetFirstPersonView(true,
false)`. (FPV drone bug: flying the drone broke both laser mods; fixed by this
reorder in `RestoreHudAndPlayerControl`.)

### Secondary RenderTexture cameras render too dark (PostProcessing v2)

7DTD renders in **linear** color space and the main camera relies on a
PostProcessing-Stack-v2 `PostProcessLayer` (tonemapping, color grading,
auto-exposure) to map linear output to the final on-screen image. A bare
secondary `Camera` (e.g. a drone/scope feed rendering to a `RenderTexture`)
skips all of it, so its feed looks noticeably **darker and flatter** than the
normal view — even in daylight. Copying `cullingMask`/`clearFlags`/`allowHDR`
is not enough.

Fix: clone the main camera's `PostProcessLayer` onto the secondary camera.
Reference `Unity.Postprocessing.Runtime.dll` (in `…/Managed`), then:

```csharp
using UnityEngine.Rendering.PostProcessing;
var src = Camera.main.GetComponent<PostProcessLayer>();
// PostProcessResources lives in a private field — reuse the main camera's.
var res = (PostProcessResources)typeof(PostProcessLayer)
    .GetField("m_Resources", BindingFlags.NonPublic | BindingFlags.Instance)
    .GetValue(src);
var layer = camGo.AddComponent<PostProcessLayer>();
layer.Init(res);
layer.volumeLayer = src.volumeLayer;          // else no volumes apply
layer.volumeTrigger = camGo.transform;         // sample volumes at the cam
layer.antialiasingMode = src.antialiasingMode;
```

The grading/tonemapping is a **global** volume, so any `volumeTrigger` picks it
up; using the camera's own transform also respects local post volumes it moves
through. (FPV drone feed darkness bug, fixed this way.)

### Anchoring first-person held-item visuals (lasers, beams, muzzle effects)

`EntityPlayerLocal` exposes public fields for the FP camera:

- `cameraTransform` (`Transform`) — the first-person camera transform. Its
  position and `right`/`up`/`forward` basis already include **view bob, weapon
  sway, and the exact pitch pivot**. The held FP item model is parented under
  this camera.
- `playerCamera` / `finalCamera` (`Camera`), `cameraContainerTransform`,
  `ModelTransform` (`Transform`), `vp_FPCamera` (property) are also public.

To make a visual *emit from the held item* (e.g. a laser starting at the
flashlight), build the start point from `cameraTransform`:

```csharp
Transform cam = localPlayer.cameraTransform; // null-check; cast holder as EntityPlayerLocal
Vector3 start = cam.position + cam.right * R + cam.up * U + cam.forward * F;
```

Do **not** derive it from `getHeadPosition()` + a basis built off
`GetLookVector()`: that uses the logical head bone, which is a *different pivot
with no view bob*, so the visual drifts vertically when pitching and jitters
when walking/running. (Airstrike designator laser bug, fixed this way.)
Remember to subtract `Origin.position` before handing the world point to a
`LineRenderer`/GameObject — see [Entity Positions](Entity Positions.md).

**Coordinate space:** `cameraTransform.position` is a Unity `Transform.position`
— already render-space (origin-relative). `getHeadPosition()` returns un-shifted
**world** coords. Don't mix them: to use the camera position in world-space math
(e.g. a `Voxel.Raycast`, which works in un-shifted world space), add
`Origin.position`; to feed it to a world-space `LineRenderer`, leave it as-is.

**Crosshair alignment:** to make a painted point / raycast land under the center
crosshair at every pitch, cast from `cameraTransform.position (+Origin.position)`
along `cameraTransform.forward` — *not* `getHeadPosition()`/`GetLookVector()`.
The head bone sits above/behind the camera pivot, so at steep pitch the parallax
pushes a near hit point visibly off the crosshair.

**Following the held-item model's own sway:** `cameraTransform` carries the
camera *view-bob*, but the FP weapon/item model has additional procedural sway
layered on top. To anchor a visual to the actual model (so it follows that extra
sway), use `EntityAlive.inventory.GetHoldingItemTransform()` (public, returns the
live held-item `Transform`; render-space, so `+Origin.position` like the camera).
Null during holster/draw — fall back to `cameraTransform`. A good pattern: base
the *position* on the model transform but keep the *offset basis* from
`cameraTransform` (right/up/forward) so screen-relative tuning stays intuitive.
Related: `ItemInventoryData.model` (`Transform`) and
`Inventory.lastdrawnHoldingItemTransform`.
