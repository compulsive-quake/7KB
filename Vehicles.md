# Vehicles

Part of the [7DTD Modding Knowledgebase](README.md). Covers custom drivable
vehicles — entity setup, the `vehicles.xml` schema, the mount/dismount
pipeline, the camera/pose system, and how to ride a kinematic prefab with
fully scripted locomotion (e.g. a hover bike or flying mount).

---

## Two routes for a custom vehicle

### Route A — vanilla physics, XML only

Extend `EntityVehicle` (or a vanilla subclass like `EntityMotorcycle`,
`EntityBicycle`, `EntityCar`) and configure all behavior in
`Config/vehicles.xml`. The wheel colliders, motors, force entries,
brakes, and steering are driven by the engine. No C# required.

```xml
<entity_class name="vehicleMyBike">
    <property name="Class" value="EntityMotorcycle"/>
    <property name="Tags" value="vehicle,usesGas"/>
    <property name="Prefab" value="#@modfolder:Resources/MyBike.unity3d?MyBikePrefab.prefab"/>
    <property name="LootListAlive" value="myBikeStorage"/>
    <property name="RotateToGround" value="true"/>
</entity_class>
```

This is the right path for *any* vehicle whose locomotion fits the
vanilla wheel-and-engine model — even "flying" ones can be approximated
with `inputUp` / `inputDown` force entries (see the **Forces** section).

### Route B — scripted locomotion on a kinematic rigidbody

Extend `EntityVehicle` and override `OnUpdateLive` to drive the bike
yourself via `SetPosition` / `SetRotation`. The vehicle still uses the
vanilla activation, attach, storage, and camera systems — you only
replace the per-tick physics. Required when the prefab has no wheel
colliders, or when locomotion is non-physical (hover, teleport, rails).

The rest of this document covers what you need to know to make either
route work — Route B has more hooks because you're carrying more weight
yourself.

---

## entityclasses.xml — required properties

```xml
<append xpath="/entity_classes">
    <entity_class name="vehicleMyBike">
        <property name="Class" value="EntityVehicle"/>     <!-- or EntityMotorcycle / Custom, NS-qualified -->
        <property name="Tags" value="vehicle,usesGas"/>
        <property name="Prefab" value="#@modfolder:Resources/MyBike.unity3d?MyBikePrefab.prefab"/>
        <property name="ModelType" value="Standard"/>
        <property name="SurfaceCategory" value="metal"/>
        <property name="IgnoreTrigger" value="true"/>
        <property name="LootListAlive" value="myBikeStorage"/>  <!-- REQUIRED, see Storage -->
        <property name="IsEnemyEntity" value="false"/>
        <property name="RotateToGround" value="true"/>          <!-- align bike up-vector to terrain -->
        <property name="MapIcon" value="myBike_UI"/>
        <property name="NavObject" value="myBike_UI"/>
    </entity_class>
</append>
```

**Custom C# class via `Class="MyEntity, MyAssemblyName"`:** the type must
be in the global namespace (no `namespace` block) so `Type.GetType` can
resolve `"MyEntity, MyAssemblyName"` without an explicit `AssemblyResolve`
hook. If you need a namespace, register an `AppDomain.CurrentDomain.AssemblyResolve`
handler in `IModApi.InitMod` that returns your loaded assembly when the
runtime asks for it by short name.

`LootListAlive` is **not optional** — `EntityVehicle.Init` does
`LootContainer.GetLootContainer(GetLootList()).size` unconditionally and
NPEs at spawn if the lookup misses. Either define your own loot table or
fall back to a vanilla one (`vehicle4x4Truck` is 10×9, `vehicleMotorcycle`
is 6×4).

---

## vehicles.xml — schema reference

`vehicles.xml` defines the *behavior* of a vehicle (parts, motors, seats,
camera). Each `<vehicle>` element's `name` must match the
`entity_class/@name`. Properties live at vehicle scope; **parts** are
nested `<property class="...">` blocks each with their own
`<property name="class" value="..."/>` discriminator.

```xml
<append xpath="/vehicles">
    <vehicle name="vehicleMyBike">
        <!-- Vehicle-scope tuning -->
        <property name="cameraDistance" value="3, 4.5"/>
        <property name="cameraTurnRate" value=".2, .35"/>
        <property name="upAngleMax" value="70"/>
        <property name="upForce" value="1"/>
        <property name="steerRate" value="120"/>
        <property name="steerCenteringRate" value="90"/>
        <property name="tiltAngleMax" value="20"/>
        <property name="motorTorque_turbo" value="5000, 2500, 8000, 4000"/>
        <property name="velocityMax_turbo" value="14, 8, 20, 10"/>
        <property name="brakeTorque" value="10000"/>
        <property name="recipeName" value="myBikePlaceable"/>

        <!-- Parts (each declares its class explicitly) -->
        <property class="chassis">
            <property name="class" value="Chassis"/>
            <property name="paint" value="M/Body"/>
        </property>

        <property class="seat0"> ... </property>
        <property class="engine"> ... </property>
        <property class="fuelTank"> ... </property>
        <property class="handlebars"> ... </property>
        <property class="motor0"> ... </property>
        <property class="force0"> ... </property>
        <property class="wheel0"> ... </property>
        <property class="headlight"> ... </property>
        <property class="storage"> ... </property>
    </vehicle>
</append>
```

### Vehicle-scope properties

| Property | Format | Purpose |
|---|---|---|
| `cameraDistance` | `near, far` | Min/max chase camera distance |
| `cameraTurnRate` | `slow, fast` | Camera follow lerp speeds |
| `upAngleMax` | float | Max body pitch from world up before correction |
| `upForce` | float | Strength of up-righting torque |
| `steerRate` | float | Degrees/sec the wheel rotates toward input |
| `steerCenteringRate` | float | Degrees/sec the wheel returns to center |
| `tiltAngleMax` | float | Lean angle at full speed |
| `tiltDampening` / `tiltThreshold` / `tiltUpForce` | float | Lean controller |
| `motorTorque_turbo` | `fwd, back, fwdTurbo, backTurbo` | Engine torque |
| `velocityMax_turbo` | `fwd, back, fwdTurbo, backTurbo` | Speed caps |
| `brakeTorque` | float | Brake strength |
| `hopForce` | `vertical, horizontal` | Bunny-hop impulse |
| `unstickForce` | float | Auto-unstick when wedged |
| `waterDrag_y_velScale_velMaxScale` | `y, scale, maxScale` | Water resistance |
| `recipeName` | string | Recipe that scrap/repair UI shows |
| `hornSound` | string | Sound name played on horn |

Anything you omit defaults to zero / disabled, so a "no engine" build is
just leaving `engine`/`motor*`/`force*`/`wheel*` out.

---

## Seats — `seat0`, `seat1`, …

```xml
<property class="seat0">
    <property name="class" value="Seat"/>
    <property name="pose" value="30"/>                       <!-- vehiclePoseHash; see Poses -->
    <property name="position" value="0, 0.5, -0.13"/>        <!-- local offset on the bike -->
    <property name="rotation" value="22.6, 0, 0"/>           <!-- local euler -->
    <property name="IKHandLPosition" value="-0.19, 0.26, -0.18"/>
    <property name="IKHandLRotation" value="3.1, 92.9, 0.2"/>
    <property name="IKFootLPosition" value="0, 0, 0"/>
    <property name="IKFootRPosition" value="0, 0, 0"/>
    <property name="exit" value="-.8,0,0 ~ .8,0,0 ~ 0,0,-1.2 ~ 0,0,1.2 ~ 0,1.3,0"/>
</property>
```

- **`position` / `rotation`** — the rider's *ModelTransform* local offset
  inside the vehicle. RootTransform always sits at the vehicle origin;
  the seat displaces only the visible body.
- **`pose`** — integer index into the `vehiclePoseHash` animator
  parameter (see **Seat Poses** below).
- **`IK*Position` / `IK*Rotation`** — solver targets for the rider's
  hands and feet, applied via `SetIKTargets` on attach.
- **`exit`** — `~`-separated list of relative dismount candidates
  `x,y,z`. The engine raycasts each in order and picks the first that
  isn't blocked. Always include an "above" candidate (e.g. `0,1.5,0`) so
  the player can never get stuck inside terrain.

### Seat poses (`vehiclePoseHash`)

`Vehicle.GetSeatPose(slot)` returns the integer from the seat's `pose`
property, then `EntityAlive.SetVehiclePoseMode(int)` writes it to the
animator's `vehiclePoseHash` parameter. Common vanilla values:

| `pose` | Pose |
|---|---|
| `0`  | Standing (unposed) |
| `20` | Lean-forward (speeder, dirt bike) |
| `30` | Motorcycle handlebars (upright with hands forward) |
| `40` | Bicycle |
| `50` | Truck/4x4 driver |
| `60` | Passenger |

If the rider looks like they're falling, T-posing, or running on the
spot when seated, the symptom is almost always *the player was never
fed through the vanilla attach pipeline* — `vehiclePoseHash` was never
set, so the animator stays in its locomotion state. See
**Mount/dismount pipeline** below.

---

## Engine, fuel, handlebars

```xml
<property class="engine">
    <property name="class" value="Engine"/>
    <property name="fuelKmPerL" value=".1"/>
    <property name="foodDrain" value=".002,.00811"/>
    <property name="gear1" value="500,2500, -1400,800,0, 700,2200,900, sound/accel1, sound/decel, ..."/>
    <property name="gear2" value="..."/>
    <property name="sound_start" value="Vehicles/MyBike_start"/>
    <property name="sound_shut_off" value="Vehicles/MyBike_shutoff"/>
</property>

<property class="fuelTank">
    <property name="class" value="FuelTank"/>
    <property name="capacity" value="600"/>
</property>

<property class="handlebars">
    <property name="class" value="Steering"/>
    <property name="transform" value="M/SteeringWheel"/>
    <property name="steerAngle" value="0, 0, 0"/>
    <!-- IK target for the rider's hands while gripping the bars -->
    <property name="IKHandLPosition" value="-0.19, 0.26, -0.18"/>
    <property name="IKHandLRotation" value="3.1, 92.9, 0.2"/>
    <property name="IKHandRPosition" value="0.19, 0.26, -0.17"/>
    <property name="IKHandRRotation" value="359.5, 85.8, 180.5"/>
</property>
```

Omit any of these to skip the part — a no-engine bike is valid and just
won't burn fuel or have an idle/run sound. `Vehicle.GetIKTargets(slot)`
collects from `handlebars`, `pedals`, and the seat itself; all are
optional.

---

## Motors and forces — XML-driven physics

This is how Route A vehicles get their motion. Motors generate `rpm`
on a chosen axis from an input trigger; forces convert motor RPM (or
direct input) into Unity rigidbody impulses.

### Motor

```xml
<property class="motor0">
    <property name="rpmAccel_min_max" value=".002, .05"/>
    <property name="rpmMax" value="3"/>
    <property name="rpmDrag" value=".993"/>
    <property name="trigger" value="relative"/>          <!-- inputForward, inputUp, inputDown, etc. -->
    <property name="axis" value="1"/>                    <!-- 0=x, 1=y, 2=z -->
</property>
```

### Force

```xml
<property class="force0">
    <property name="trigger" value="motor0"/>            <!-- or inputUp, inputDown, inputStrafe -->
    <property name="type" value="relative"/>             <!-- or relativeTorque -->
    <property name="force" value="0, .07, 0"/>           <!-- per-axis force vector -->
    <property name="ceiling" value="10, 15"/>            <!-- optional: speed at which force tapers -->
</property>
```

### Recipe — vertical thrust on a wheeled vehicle

```xml
<property class="force2">
    <property name="trigger" value="inputUp"/>           <!-- mapped to RShift in vanilla controls -->
    <property name="type" value="relative"/>
    <property name="force" value="0, 0.5, 0"/>
</property>
<property class="force4">
    <property name="trigger" value="inputDown"/>         <!-- mapped to LCtrl -->
    <property name="type" value="relative"/>
    <property name="force" value="0, 0, -.4"/>
</property>
```

The vanilla `EntityMotorcycle` + a positive `inputUp` force is enough to
build a "flying" vehicle without any C#. Pair with `RotateToGround=true`
(or `false` if you want free pitch) and tuned `upForce` / `upAngleMax`.

### Wheels

```xml
<property class="wheel0">
    <property name="steerTransform" value="M/frontWheelStear_joint"/>
    <property name="tireTransform" value="M/frontWheelStear_joint/frontWheel_joint"/>
    <property name="tireSuspensionPercent" value="1"/>
    <property name="torqueScale_motor_brake" value="1, 1"/>
</property>
```

`steerTransform` (optional) is the joint that yaws on steering.
`tireTransform` is the joint that spins. Both reference paths inside the
vehicle's prefab (typically rooted at `M/`).

---

## Mount/dismount pipeline

This is the most-misunderstood part of building a custom vehicle in C#.
The pipeline that makes the rider seated, posed, IK-rigged, and viewed
in third-person is **not** triggered by `EntityAlive.SetPosition` —
parking the player on the bike with `SetPosition` each tick gives you
the "rider falling off a cliff" animation because the animator never
learns it should be in a vehicle pose.

### The flow

```
vehicle.EnterVehicle(player)
    └─ player.StartAttachToEntity(vehicle, slot=-1)
        └─ (network round-trip on client)
            └─ player.AttachToEntity(vehicle, slot)               // EntityPlayerLocal override
                ├─ SetFirstPersonView(false, lerp:true)           // camera switch
                ├─ vp_FPController.Player.Driving.Start()
                └─ base.AttachToEntity(vehicle, slot)             // EntityAlive
                    ├─ vehicle.AttachEntityToSelf(player, slot)   // YOUR override
                    │   ├─ vehicle.GetSeatPose(slot)              // reads XML pose
                    │   ├─ player.SetVehiclePoseMode(pose)        // sets vehiclePoseHash
                    │   ├─ player.transform.gameObject.layer = 24
                    │   ├─ player.m_characterController.Enable(false)
                    │   ├─ player.SetIKTargets(GetIKTargets(slot))
                    │   ├─ SetVehicleDriven()                     // flips vehicleRB.isKinematic = false
                    │   └─ CameraInit()                           // m_Current3rdPersonBlend = 1f
                    ├─ RootTransform.SetParent(vehicle.transform)
                    ├─ RootTransform.localPosition = Vector3.zero
                    ├─ ModelTransform.localPosition = seat.enterPosition
                    └─ AttachedToEntity = vehicle
```

### Your custom vehicle's responsibility

In a custom `EntityVehicle` subclass, the minimum you need:

```csharp
public override bool OnEntityActivated(int idx, Vector3i tePos, EntityAlive activator)
{
    if (/* ride command */) {
        EnterVehicle(activator);   // hand off — the pipeline does the rest
        return true;
    }
    // ... storage, pickup, etc.
}

public override int AttachEntityToSelf(Entity _entity, int slot = -1)
{
    int s = base.AttachEntityToSelf(_entity, slot);
    if (s >= 0 && /* you use a kinematic RB */) {
        vehicleRB.isKinematic = true;   // base.SetVehicleDriven flipped it off
        RBActive = false;
    }
    return s;
}

public override void DetachEntity(Entity _entity)
{
    base.DetachEntity(_entity);
    if (/* you use a kinematic RB */) {
        vehicleRB.isKinematic = true;
    }
}
```

Dismount is `rider.SendDetach()` — that routes through `Detach()` which
re-enables the character controller, clears `vehiclePoseHash`, removes
IK targets, restores first-person view, and uses `seat.exit` to find a
safe dismount position.

### The `SetVehicleDriven` gotcha

`EntityVehicle.AttachEntityToSelf` calls `SetVehicleDriven()` which
sets `vehicleRB.isKinematic = false`, sets layer 21, and adds the
rigidbody back to Unity's physics solver. If your vehicle uses scripted
locomotion (e.g. you call `SetPosition` each tick), Unity will then
fight you — gravity/drag/inertia will cause the bike to drift, and your
rigidbody might be at the wrong position by the time the next frame's
script integration runs. **Force `vehicleRB.isKinematic = true` after
calling `base.AttachEntityToSelf`.**

`SetVehicleDriven` is non-virtual in the base class, so you can't
override it directly — you patch the symptom in your `AttachEntityToSelf`
override instead.

### Network / multiplayer note

`StartAttachToEntity` sends a packet to the server and waits for the
`AttachClient` packet to come back. On a host or in single-player it
runs synchronously; as a client there's a one-frame delay between
"player pressed E" and `AttachEntityToSelf` firing. If you track the
rider in your own field, set it inside the `AttachEntityToSelf` override
(not in the activation handler), so it's only set after the server
confirms the seat is yours.

---

## AttachedToEntitySlotInfo — what the engine asks the vehicle for

When the player attaches, the engine calls `vehicle.GetAttachedToInfo(slot)`
to learn how to position the rider, restrict their look, and where to
dismount them. The default `EntityVehicle` implementation reads
`seat0.position`, `seat0.rotation`, `seat0.exit` from XML and returns:

```csharp
new AttachedToEntitySlotInfo {
    bKeep3rdPersonModelVisible = true,
    bReplaceLocalInventory     = true,
    pitchRestriction = new Vector2(-30f, 30f),    // look down/up
    yawRestriction   = new Vector2(-90f, 90f),    // look left/right
    enterParentTransform = vehicle.transform,
    enterPosition = parsed seat XML,
    enterRotation = parsed seat XML,
    exits = parsed seat XML,
};
```

Override `GetAttachedToInfo` if you want different look restrictions,
re-parent the rider somewhere besides `vehicle.transform`, or compute
the seat position dynamically (e.g. moving turret).

---

## Custom prefab without a Physics child — required scaffolding

Vanilla vehicle prefabs ship with a `Physics` child holding a
`Rigidbody` and `WheelCollider`s. If your custom prefab doesn't, you
must synthesize the structure before `base.Init` runs or you'll NPE
inside `EntityVehicle.Init` and `PostInit` on the missing
`PhysicsTransform` / `vehicleRB`:

```csharp
public override void Init(int _entityClass)
{
    if (PhysicsTransform == null && RootTransform != null)
    {
        var physGO = new GameObject("Physics");
        physGO.transform.SetParent(RootTransform, worldPositionStays: false);
        var rb = physGO.AddComponent<Rigidbody>();
        rb.isKinematic = true;
        rb.useGravity = false;
        rb.mass = 150f;
        PhysicsTransform = physGO.transform;
    }
    base.Init(_entityClass);
}
```

The activation-system tagging that `EntityVehicle.Init` does (recursive
`E_Vehicle` tag on the `PhysicsTransform` subtree) will only cover the
empty stub — the visible mesh under `ModelTransform` is unreached. Add
your own pass on first update:

```csharp
void EnsureVehicleColliderHook()
{
    if (_installed) return;
    var modelRoot = emodel?.GetModelTransform();
    if (modelRoot == null) return;   // model loads async; retry next tick

    Utils.SetTagsIfNoneRecursively(modelRoot, "E_Vehicle");

    var ccf = transform.gameObject.GetComponent<CollisionCallForward>()
              ?? transform.gameObject.AddComponent<CollisionCallForward>();
    ccf.Entity = this;

    _installed = true;
}
```

Without the `E_Vehicle` tag the player's interaction raycast hits an
untagged collider and the activation prompt never appears. The
`CollisionCallForward` routes child-collider hits back to the entity so
the engine knows what was clicked.

### Skip the inherited FixedUpdate

`EntityVehicle.FixedUpdate` runs `PhysicsFixedUpdate` which dereferences
`vehicleRB` for wheel torques and brakes. Your kinematic stub has no
wheels and no torque to apply, but the dereferences still run. Hide the
parent's `FixedUpdate` with a `new` no-op (Unity's message dispatcher
calls the most-derived definition):

```csharp
new void FixedUpdate() { }
```

---

## RootTransform, ModelTransform, PhysicsTransform — what's what

- **`RootTransform`** = `entity.transform`. The world-space anchor.
  When the engine moves an entity, it moves this. The rider is
  `SetParent`'d here on attach.
- **`ModelTransform`** = the visible mesh. Lives under `RootTransform`.
  This is what you scale, rotate-for-tilt, or hide. The seat
  `enterPosition` is applied as `ModelTransform.localPosition` on the
  *rider's* model, not the vehicle's.
- **`PhysicsTransform`** = the rigidbody owner. Lives under
  `RootTransform`. For wheeled vehicles it holds the chassis collider,
  wheel colliders, and CollisionCallForward.

`SetPosition(pos)` on `EntityVehicle` updates *both* `RootTransform`
and `ModelTransform` (so the parented rider follows automatically) but
leaves the rigidbody to Unity if non-kinematic.

---

## Activation commands

Vehicles use the standard `EntityActivationCommand[]` pattern but are
typically state-driven — different commands when free vs. ridden:

```csharp
static readonly EntityActivationCommand[] _availableCmds = {
    new EntityActivationCommand("Ride",    "interact",       true),
    new EntityActivationCommand("Storage", "loot_sack",      true),
    new EntityActivationCommand("Pickup",  "open_inventory", true),
};
static readonly EntityActivationCommand[] _ridingCmds = {
    new EntityActivationCommand("Dismount", "interact",  true),
    new EntityActivationCommand("Storage",  "loot_sack", true),
};
static readonly EntityActivationCommand[] _occupiedCmds = new EntityActivationCommand[0];

public override EntityActivationCommand[] GetActivationCommands(Vector3i p, EntityAlive focusing)
{
    if (AttachedMainEntity == null) return _availableCmds;
    if (focusing != null && focusing.entityId == AttachedMainEntity.entityId) return _ridingCmds;
    return _occupiedCmds;   // bystanders see nothing
}
```

The icon strings (`"interact"`, `"loot_sack"`, `"open_inventory"`) come
from the engine's icon set — see vanilla vehicle entities for the full
list.

---

## Storage — VehicleInventory and the auto-allocated TileEntity

Vehicles get loot containers for free as long as `LootListAlive` resolves.
Open the storage UI from your activation handler:

```csharp
bool OpenStorage(EntityPlayerLocal player)
{
    var te = world.GetTileEntity(entityId)?.GetSelfOrFeature<ITileEntityLootable>();
    if (te == null) return false;

    var ui = LocalPlayerUI.GetUIForPlayer(player);
    if (ui == null) return false;

    te.bWasTouched = te.bTouched;
    te.bTouched = true;

    ui.windowManager.CloseAllOpenWindows();
    string label = Localization.Get(EntityClass.list[entityClass].entityClassName);
    var window = ui.windowManager.GetWindow("looting");
    ((XUiC_LootWindowGroup)((XUiWindowGroup)window).Controller).SetTileEntityChest(label, te);
    ui.windowManager.Open("looting", _bModal: true);
    return true;
}
```

The `TileEntityLootContainer` is allocated lazily on first
`world.GetTileEntity(entityId)` call because `LootListAlive` is set on
the entity class. You don't need to instantiate it yourself.

The `storage` part on the vehicle adds a *modular* storage slot —
expanding capacity when the player installs a "storage" mod into the
vehicle's part inventory:

```xml
<property class="storage">
    <property name="class" value="Storage"/>
    <property name="mod" value="storage"/>
</property>
```

`UpdateStorageModCount(int)` then bumps the bag's row count.

---

## Pickup — refunding the placeable item

Mirror the vanilla minecart/bicycle pattern: refund one placeable item
into the player's inventory and despawn the bike.

```csharp
bool TryPickup(EntityPlayerLocal player)
{
    int itemId = ItemClass.GetItem("myBikePlaceable", false).type;
    if (itemId == 0) return false;

    var stack = new ItemStack(new ItemValue(itemId, false), 1);
    if (!player.bag.AddItem(stack) && !player.inventory.AddItem(stack))
    {
        GameManager.Instance.ItemDropServer(stack, player.GetPosition(),
            new Vector3(0.5f, 0f, 0.5f), player.entityId);
        player.PlayOneShot("itemdropped");
    }
    world.RemoveEntity(entityId, EnumRemoveEntityReason.Killed);
    return true;
}
```

---

## Per-tick locomotion (Route B)

A scripted bike runs all its motion out of `OnUpdateLive`:

```csharp
public override void OnUpdateLive()
{
    base.OnUpdateLive();
    EnsureVehicleColliderHook();

    EntityPlayerLocal rider = ResolveRider();   // via AttachedMainEntity / your own field
    if (rider == null) {
        // Bleed off speed and let the bike sit.
        ApplyIntegratedMotion();
        return;
    }

    if (!rider.isEntityRemote)
        HandleRiderInput(rider);                // direct Input.GetKey is fine — see below

    ApplyIntegratedMotion();                    // SetPosition + SetRotation per tick
}
```

### Input while attached

When a player is attached, `vp_FPController.Player.Driving.Start()` is
running and the engine inserts a `PlayerActionsVehicle` action set on
top of the input stack. *That action set re-routes WASD into the
vehicle's `movementInput`* — a Route A vehicle reads
`movementInput.MoveWorld` from there.

For a Route B vehicle that polls `Input.GetKey(KeyCode.W)` directly:
direct Unity input bypasses the action-set system entirely, so WASD
still register. But pay attention to:

- **Space**: in driving mode it triggers the vehicle's hop. If you also
  want it as a dismount key, gate on `Input.GetKeyDown(KeyCode.Space)`
  *before* the Driving system consumes it (your `OnUpdateLive` runs
  per-frame, your handler still sees the GetKeyDown).
- **Left Ctrl / Right Shift**: free for vertical thrust — the vehicle
  action set doesn't claim them.
- **A / D**: vanilla vehicle uses these for steering input. If you read
  them directly *and* you have steering wheels in XML, both systems
  fire. Use empty steering setup (`steerAngleMax="0"`) on a Route B
  vehicle to keep the vanilla path silent.

### Anti-tunneling

Clamp per-tick displacement so a long frame can't tunnel through
terrain. A 50ms tick at 30 m/s = 1.5m of motion per axis — clamp
slightly above:

```csharp
displacement.x = Mathf.Clamp(displacement.x, -1.5f, 1.5f);
displacement.y = Mathf.Clamp(displacement.y, -1.5f, 1.5f);
displacement.z = Mathf.Clamp(displacement.z, -1.5f, 1.5f);
```

### Soft floor against terrain

`World.GetHeight((int)Mathf.Floor(x), (int)Mathf.Floor(z))` returns the
top-of-block Y as a float. Cheaper and smoother than `Voxel.GetCappedY`:

```csharp
float ground = world.GetHeight((int)Mathf.Floor(nextPos.x),
                               (int)Mathf.Floor(nextPos.z));
float floor = ground + 1.2f;     // hover height above the surface
if (nextPos.y < floor) {
    nextPos.y = floor;
    if (_verticalSpeed < 0f) _verticalSpeed = 0f;
}
```

---

## Disabling collision response

A scripted bike doesn't want to ragdoll on touch — return false from
`CanCollideWith`:

```csharp
public override bool CanCollideWith(Entity _other) => false;
```

Don't disable `nativeCollider` outright — the *interaction* raycast
still needs it for the activation prompt.

---

## Decompiling vanilla source for reference

Most of the seat/attach/camera plumbing in this doc came from reading
`Assembly-CSharp.dll` directly. Use any IL decompiler (`ilspycmd`,
`dnSpy`, `ILSpy`):

```bash
ilspycmd "<7DTD>/7DaysToDie_Data/Managed/Assembly-CSharp.dll" -o ./decompiled
```

The shipped DLL is publicized — members marked
`[PublicizedFrom(EAccessModifier.Private)]` or `Protected` were
private/protected in the original source but exposed as public for
modders. Treat them as public, but assume they may move or rename
between game patches.

Fast greps in the decompiled output that pay off when working on
vehicles:

| Looking for | Grep |
|---|---|
| Mount entry point | `EnterVehicle\b` |
| Rider attach hook | `AttachEntityToSelf\b` |
| Detach hook | `DetachEntity\b` / `Detach\(\)` |
| Pose mapping | `SetVehiclePoseMode` / `vehiclePoseHash` |
| Camera 3rd-person flip | `m_Current3rdPersonBlend` |
| Seat XML reader | `GetSeatPose` / `GetIKTargets` |
| Slot info | `AttachedToEntitySlotInfo` / `GetAttachedToInfo` |
| Force / motor parser | `SetupForces` / `SetupMotors` / `SetupWheels` |

---

## Common symptoms → root cause

| Symptom | Likely cause |
|---|---|
| Rider is in falling/T-pose on the bike | Mount didn't go through `EnterVehicle` → `vehiclePoseHash` was never set |
| Camera stays first-person | `AttachEntityToSelf` never ran → `CameraInit` never called. Check that `EnterVehicle` got past the network round-trip (server ack) |
| Bike drifts off after mount | `SetVehicleDriven` flipped `vehicleRB.isKinematic` to false. Force it back to true in your `AttachEntityToSelf` override |
| Activation prompt never appears | `E_Vehicle` tag not applied to the visible mesh — only to the empty `Physics` stub. Tag the `ModelTransform` subtree |
| NPE in `EntityVehicle.Init` | `LootListAlive` resolves to nothing, or `PhysicsTransform`/`vehicleRB` is null. See **Custom prefab** scaffolding |
| Rider stuck inside the ground on dismount | All `seat.exit` candidates are blocked. Add an "above" exit (`0,1.5,0`) |
| WASD does nothing while riding | Route A vehicle with no `motor*` or `force*` entries, OR Route B vehicle whose `OnUpdateLive` returns before the input handler runs |
| Bike falls under gravity when nobody's on it | Custom prefab with no `useGravity = false` on the synthesized rigidbody |

---

## Related

- [Entities](Entities.md) — base entity class, AI tasks, drops
- [Asset Bundles](Asset%20Bundles.md) — loading the bike prefab
- [XML Patching (XPath)](XML%20Patching%20(XPath).md) — how to `<append>` to `vehicles.xml`
