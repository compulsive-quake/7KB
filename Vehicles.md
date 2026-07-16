# Vehicles

Part of the [7DTD Modding Knowledgebase](README.md). Covers custom drivable
vehicles — entity setup, the `vehicles.xml` schema, the mount/dismount
pipeline, the camera/pose system, and how to ride a kinematic prefab with
fully scripted locomotion (e.g. a hover bike or flying mount).

---

## ⚠️ Game update ~2026-06-29 (Steam buildid 23906531) — Entity API break

A Steam update rewrote the entity init + activation APIs. Symptom: mods that
compiled before suddenly fail with **CS0115 "no suitable method found to
override"**. Verified while porting Oppressor (2026-07-09). Changes:

- **`Entity.Init(int)` → `Init(int _entityClass, EntityInstanceAssets _assets,
  EModelInstanceAssets _eModelAssets)`.** The model prefab now arrives via
  `_assets`; `EntityFactory` still assigns `RootTransform`/`ModelTransform`
  *before* calling `Init`, so pre-`base.Init` scaffolding still works.
- **Activation commands are now id/callback-based**, replacing the
  index-array system:
  - `GetActivationCommands(Vector3i, EntityAlive)` is **gone**. Base command
    sets are declared in `InitLocalActivationCommands(Action<EntityActivationCommand> _addCallback)`
    (EntityVehicle registers `drive, ride, service, repair, lock, unlock,
    storage, keypad, refuel, take, horn` by string id).
  - Enable/disable logic lives in
    `AllowActivationCommand(ReadOnlySpan<char> _commandName, EntityPlayerLocal _playerFocusing)`
    — compare with the inherited helper `CommandIs(span, "drive")`.
  - `OnEntityActivated(int, Vector3i, EntityAlive)` →
    `void OnEntityActivated(EntityActivationCommand _command, EntityPlayerLocal _playerFocusing)`
    (returns void, dispatch on `_command.commandId`).
  - `EntityActivationCommand` is now a struct with `commandId`, `icon`,
    `commandText` (already localized via `entitycommand_<id>` at ctor time),
    `enabled`, `activateTime`. To relabel a base command, override
    `InitLocalActivationCommands`, wrap the callback, and rewrite
    `cmd.commandText` on the struct copy before forwarding.
  - The `ReadOnlySpan<char>` parameter forces the mod to compile against the
    **game's own mscorlib** — see the BCL-reference note in
    [Mod Structure](Mod%20Structure.md).
- **`Entity.PhysicsInit` destroys a direct `RootTransform` child *named*
  `"Physics"`** when it also finds a `"Physics"`-*tagged* transform under
  `ModelTransform` (it assumes a stale duplicate; logs "has old Physics").
  `Entity.InitCommon` tags any pre-seeded `PhysicsTransform` with the
  `"Physics"` tag, so a synthesized physics stub must **not** be named
  `"Physics"` or it gets destroyed at end of frame → `vehicleRB` goes null.
  Name it anything else (e.g. `"OppressorPhysics"`) and pre-assign
  `PhysicsTransform` before `base.Init`.
- **`EModelBase.DoRagdoll`** grew a leading `RagdollMode` enum param:
  `DoRagdoll(RagdollMode _mode, float stunTime, EnumBodyPartHit, Vector3
  forceVec, Vector3 forceWorldPos, bool isRemote)` (`RagdollMode.Default`
  matches old behavior).
- **Vehicle storage UI**: `XUiC_LootWindowGroup.SetTileEntityChest` and
  `GUIWindowManager.CloseAllOpenWindows` are gone. The base "storage"
  activation command opens the vehicle `bag` via
  `LockManager.Instance.LockRequestLocal(this, new EntityLockContext("storage", bag), 0)`
  — don't hand-roll the looting window anymore.
- `EntityAlive` now declares a non-virtual `LateUpdate()`; a scripted-motion
  `LateUpdate` in a subclass needs the `new` keyword (CS0108 otherwise).

The code samples below have been updated to the new API where marked; older
snippets that still show `Init(int)` / index-based activation are pre-update.

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

## Combined vehicle prefab contract (Route A with your own physics)

Verified against decompiled `EntityFactory`/`EntityVehicle`/`Vehicle` (V 2.x,
2026-07, built for the Supra mod). If you ship a full custom prefab via the
entity `Prefab` property (instead of only overriding `Mesh` on a vanilla
vehicle), the engine expects this exact structure:

```
MyCarP (root)                  ← becomes RootTransform
  Physics                      ← PhysicsTransform = prefabT.Find("Physics")
    Rigidbody                  ← vehicleRB (engine forces useGravity=false and
                                  applies its own -9.81*mass each FixedUpdate;
                                  set automaticCenterOfMass=false + explicit low
                                  centerOfMass for a car or Init overwrites it
                                  with (0, 0.1, 0))
    Wheel0..WheelN             ← SetupWheels does PhysicsTransform.Find("Wheel"+i)
                                  .GetComponent<WheelCollider>() — NPEs if missing.
                                  One per wheelN part in vehicles.xml, contiguous.
    B* (BoxColliders)          ← body collision, bake layer 21 ("Physics")
  GameObject                   ← literally named "GameObject"! EntityFactory does
                                  prefabT.Find("GameObject") and, if found, makes
                                  it the entity host + ModelTransform. Vanilla
                                  vehicle prefabs all have this oddly-named child.
    M                          ← paint/part root ("M/..." paths in vehicles.xml;
                                  chassis paint="M")
    Wheel0..WheelN             ← VISUAL wheels: vehicles.xml steerTransform /
                                  tireTransform are found under
                                  Vehicle.GetMeshTransform() (= ModelTransform,
                                  or its "Mesh" child if one exists)
```

Key facts:

- **No baked tags needed** — `EntityVehicle.Init` sets `E_Vehicle` on the root
  and recursively on the Physics subtree at runtime. (Bake layer 21 on the
  Physics colliders anyway; Init only re-layers the `Physics` node itself.)
- **Wheel order = steering order.** `EntityVJeep.UpdateWheelsSteering` steers
  `wheels[0]` and `wheels[1]` — make Wheel0/Wheel1 the front axle.
- **Motor model:** `wheelC.motorTorque = input * motorTorque_turbo[i] *
  wheel.torqueScale_motor` — the XML torque is applied PER WHEEL, not split.
  RWD = `torqueScale_motor_brake` `"0, 1"` on fronts, `"1, .75"` on rears.
  Non-turbo input is additionally halved (`!running → wheelMotor *= 0.5`).
  For real-world launch tuning in normal driving, double the desired effective
  forward wheel torque in the first `motorTorque_turbo` slot; use the third
  slot for the un-halved turbo/shift torque.
  `velocityMax_turbo` is a hard horizontal-velocity clamp, so real-world top
  speeds are safe to encode directly (Supra: 69.3 m/s turbo).
- **Acceleration tuning — compare against vanilla, not reality (Supra 2026-07):**
  effective accel ≈ `XMLtorque × (0.5 if non-turbo) × Σ torqueScale_motor /
  (wheelRadius × rigidbodyMass)`. Vanilla rigidbody masses (dumped from
  `vehicles.bundle`): bicycle/minibike/motorcycle **250 kg**, gyrocopter 500,
  4x4 2400. A "realistic" car tune (1700 Nm/wheel on 1550 kg) accelerates
  *slower* than the motorcycle (1400 XML on 250 kg ≈ 7.4 m/s²) and feels dead.
  **Traction cap:** per-axle peak force ≈ `axleLoad × fwdFrictionStiffness`
  (default curve extremum 1.0); torque demand far past the peak pushes slip
  onto the friction *asymptote* (0.5 × peak) — more XML torque then makes the
  vehicle SLOWER. A 1550 kg RWD car with stiffness-2 tires caps near 10 m/s²;
  to go quicker either raise stiffness in the prefab or give the front wheels
  partial `torqueScale_motor` (rear-biased AWD, XML-only, no bundle rebuild).
- **Live vehicle tuning at runtime (Supra Tuner, 2026-07):** everything the
  motor model reads is publicized and writable per entity — `ev.GetVehicle()`
  gives `MotorTorqueForward/Backward/TurboForward/TurboBackward`,
  `VelocityMax*` (same 4), `brakeTorque`; `ev.wheels[i]` gives
  `motorTorqueScale` and the friction bases. Write grip to
  `forwardStiffnessBase`/`sideStiffnessBase`, NOT the WheelCollider —
  `SetWheelsForces` recomputes `wheelC.forwardFriction.stiffness =
  forwardStiffnessBase * frictionPercent` every physics tick and would stomp
  a direct write. Values reset on entity reload, so re-apply on attach (poll
  `player.AttachedToEntity as EntityVehicle` + entityId change in a
  MonoBehaviour). The non-turbo 0.5× input halving applies to reverse too.
  The Supra mod wires these to a zPhone app (`Supra/zphone/`, `SupraTuning`).
- **Custom radial command + icon (Supra "Paint", 2026-07):** subclass the
  vehicle entity, append in `InitLocalActivationCommands` via
  `_addCallback(new EntityActivationCommand(id, icon))`; label comes from
  Localization `entitycommand_<id>`, and the radial resolves the icon as
  UIAtlas sprite `ui_game_symbol_<icon>` — mods can ship that sprite as
  `UIAtlases/UIAtlas/ui_game_symbol_<icon>.png` (Oppressor does the same).
  Enable it in `AllowActivationCommand` (unknown ids default to disabled)
  and handle it in `OnEntityActivated`.
- **Custom car paint vs the dye system (Supra 2026-07):** to let players pick
  arbitrary colors, neutralize the paint material once
  (`mat.mainTexture = Texture2D.whiteTexture`) and write `mat.color`; get the
  material from `vehicle.mainEmissiveMat` (cached by SetColors at spawn).
  `Vehicle.SetColors` stomps `_Color` white at every spawn/load AND on every
  driver enter/leave (`AttachEntityToSelf` slot 0 calls it) — a poll re-apply
  leaves a visible white flash, so Harmony-postfix `Vehicle.SetColors`
  (`__instance.entity` is public) and re-assert in the same frame; keep a
  slow poll only as a safety net. Persist per
  `entityId` in the SAVE folder (`GameIO.GetSaveGameDir()`) — ids are
  per-save and the save dir survives redeploys, unlike the deployed mod's
  Config/. To keep vanilla dyes from fighting it, hide the vehicle window's
  cosmetics grid via a Harmony postfix on
  `XUiC_VehicleWindowGroup.CurrentVehicleEntity` setter
  (`cosmeticGrid.ViewComponent.IsVisible = false` for your class only).
- **Brake lights (Supra 2026-07):** don't use the vanilla `tailEmissive`
  headlight property — it writes `_EmissionColor` on the slot-0 PAINT
  material (`vehicle.mainEmissiveMat`), i.e. the whole body glows. Instead
  drive a dedicated taillight material: clone ONLY that slot via the
  `sharedMaterials` array (never call `renderer.materials` — it re-instances
  every slot including the game's instanced slot-0 paint material, orphaning
  `mainEmissiveMat` until the next SetColors), then flip `_EmissionColor`
  from `Vehicle.CurrentIsBreak` (set each physics tick; MP-synced via entity
  flags; gate on `HasDriver`). CRITICAL: the `_EMISSION` Standard-shader
  variant must ship in the bundle — enable the keyword on the material at
  PREFAB BUILD time (black emission = off); `EnableKeyword` at runtime alone
  renders nothing because the variant was stripped.
- **Rollover in corners (Supra 2026-07):** a car flips when lateral accel ×
  CoM height exceeds g × half-track. Side grip stiffness S lets the tires
  pull ~S·g laterally, so grippy sports tires (stiffness 2) flip a CoM that
  would be fine on vanilla tires — the Supra's baked CoM y=.35 gave a ~2.3g
  static threshold, right at the grip limit. Two fixes, both runtime (no
  bundle rebuild): lower `vehicleRB.centerOfMass` (engine only overwrites it
  when `automaticCenterOfMass` is true — see prefab contract above; re-apply
  on attach since entity reload restores the prefab value), and add
  anti-roll bars in a `FixedUpdate` (classic recipe: per axle, force =
  (travelL − travelR) × strength, push compressed corner up / extended down
  via `AddForceAtPosition`; travel from `GetGroundHit` point in wheel-local
  space). ~30% of spring rate (15000 vs springs 50000) is a good start.
  `WheelCollider` needs a `UnityEngine.VehiclesModule.dll` reference —
  it is NOT in PhysicsModule.
- **WheelCollider placement:** pivot the collider `suspensionDistance *
  targetPosition` above the visual wheel center so the loaded wheel rests at
  the visual position (vanilla 4x4: pivot 0.14 above hub, susDist .43, target .5).
- **Splitting wheels out of a Sketchfab car FBX** (two shapes seen so far):
  - *Single mesh, wheels tagged by material* (`*_WH*`): split triangles by
    material, cluster into 4 wheels by quadrant around the wheel-soup centroid.
  - *Multi-node* (Supra 2026-07, "maya2sketchfab" exports): each wheel is its
    own node but with MIXED materials (tire + rim `metal`/`GlossBlackpaint`
    shared with the body), so material-splitting would tear rim triangles off
    the body. Instead classify a NODE as a wheel iff its renderer carries a
    `tire` material, and keep brake-caliper nodes in the body so they don't
    spin with the rim.
  In both cases rebuild each wheel mesh with the pivot at its hub, and verify
  the car's front from a material centroid (bonnet/hood at +Z, or taillights /
  license-plate at -Z) before trusting the export's orientation. See
  `Supra/UnityProject/Assets/Editor/SetupSupraPrefab.cs` for the full pipeline.
- **Inspecting an unknown binary FBX before adapting the pipeline:** node and
  material names are readable by regex-scanning the binary for
  `name\x00\x01Model|Material` pairs (python, no FBX SDK), and referenced
  texture paths by scanning for `*.png|jpe?g`. For geometry (per-node bounds,
  submesh→material map, which nodes are wheels), run a throwaway editor script
  headless (`Supra/UnityProject/Assets/Editor/SupraInspectFbx.cs` pattern:
  import → instantiate → dump `NODE path verts tris center size mats=[...]` →
  `EditorApplication.Exit`). Sketchfab FBX materials are often ALL plain white
  with textures referenced from the author's absolute paths — expect to
  hardcode per-material-name colors/textures in the build script.
- **Paint/dye contract — material slot 0 under the `paint` transform gets
  STOMPED (Supra, 2026-07):** `Vehicle.SetColors` → `VehiclePart.SetColors`
  runs at every spawn/load and does `renderers[0].material.color = tint` where
  `renderers[0]` is the FIRST renderer under the chassis `paint` transform and
  the tint is the item's `TintColor` property override — **opaque white
  (1,1,1,1) when no dye is installed**. Two consequences for custom prefabs:
  1. Slot 0 of that renderer MUST be the car-paint material. If your bake
     orders submeshes alphabetically, something else lands there (the Supra's
     `balck_tint_glass` did → the game painted the windows opaque white;
     "dye applied to the windows"). Pin the paint material to slot 0 in the
     mesh build.
  2. The paint material's `_Color` is REPLACED, not multiplied — author the
     actual paint color in the albedo TEXTURE and keep `_Color` white
     (vanilla vehicles all do this: `4X4truck` etc. are white-`_Color` +
     `_MainTex` atlas). A color authored in `_Color` is lost at spawn.
  The same slot-0 material is also cached as `vehicle.mainEmissiveMat`;
  `VPHeadlight.SetTailLights` writes `_EmissionColor` on it only if the
  headlight part sets `tailEmissive`, so leave that property unset unless
  slot 0 has a proper (mostly-black) emission map. Renderers tagged `LOD`
  under the paint transform get their whole material REPLACED by the slot-0
  instance — don't tag custom child renderers `LOD` unless they should take
  paint. Verify what actually shipped by dumping renderer material order from
  the bundle with UnityPy (`MeshRenderer.m_Materials` → material names).
- **Generated mesh `.asset` files from a heavy FBX can exceed GitHub's 100 MB
  hard push limit** (Supra 2026-07: the rebuilt 1.13 M-vert body mesh
  serialized to ~105 MB and blocked `git push`). Gitignore the generated
  `Generated/*.asset` (+ `.meta`) — they're rebuilt from the FBX by the
  headless build — and if one is already in an unpushed commit, rewrite that
  commit (`git restore --staged .` → `git rm --cached <assets>` →
  `git commit --amend`); GitHub rejects the push if ANY commit in the range
  contains a >100 MB blob, not just the tip.
- Vanilla 4x4 reference numbers (dumped from `vehicles.bundle` with UnityPy):
  Rigidbody mass 2400, drag .05/.05; WheelCollider r=.63, mass 45, susDist .43,
  spring 15000/2000/.5, friction stiffness 1.8. Addressables bundles live under
  `Data/Addressables/Standalone/automatic_assets_entities/vehicles.bundle` —
  NOT in `data.unity3d`.
- `LootList` (not just `LootListAlive`) is what vanilla vehicles use for their
  storage container; the container is defined in `loot.xml` (`count="0"`,
  empty element).

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

### How speed actually happens (V2.x decompile, 2026-07)

`velocityMax` is **not** a torque falloff — it's a hard clamp, and the thing
being clamped ramps. In `EntityVehicle.FixedUpdate`:

- `motorTorque` goes straight to the Unity WheelColliders
  (`wheel.wheelC.motorTorque = wheelMotor * torque * wheel.motorTorqueScale`),
  so real acceleration is WheelCollider slip/friction — not reproducible
  outside Unity. Donor-tune cars are massively over-torqued (6000 N·m, AWD),
  so the launch is grip-limited, not torque-limited (~1 g with the donor
  prefab's forward friction stiffness 2.0).
- The **cap eases**: `velocityMax = Mathf.MoveTowards(velocityMax, target,
  (target > velocityMax ? 2.5f : 1.5f) * dt)`, then horizontal rigidbody
  velocity is scaled down to it. So engaging turbo doesn't jump the cap from
  33 → 69.3; it climbs at 2.5 m/s² and takes ~15 s.
- The target is `VelocityMaxForward` **regardless of input** — only reverse
  (`moveForward < 0`) or turbo (`running && CanTurbo && moveForward != 0`)
  swap it. So the cap sits at the forward value while you idle, and is
  already there when you press W.
- `Vehicle.AirDragVelScale` **defaults to 0.997** and multiplies rigidbody
  velocity every 50 Hz FixedUpdate = a continuous decay of −ln(.997)/.02
  ≈ 0.15/s. This, not the clamp, sets true terminal speed: a car pulling at
  ~1 g settles near 9.8/0.15 ≈ 65 m/s, so a 69.3 turbo cap is never reached.
  Omitting `airDrag_velScale_angVelScale` does not mean "no drag".

Practical: for a hold-W speed model use `dv/dt = a_grip − 0.15·v`, clamped to
the ramping `velocityMax`. AssettoCar's sounds-page drive preview
(`src-tauri/src/pipeline/drive_preview.rs`) is built on exactly this.

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
- **Live seat tuning pattern:** for a temporary tuning DLL, poll input from
  `ModEvents.UnityUpdate`, require
  `player.AttachedToEntity != null` and
  `EntityClass.GetEntityClassName(player.AttachedToEntity.entityClass)` to
  match the target vehicle, then adjust `player.ModelTransform.localPosition`
  and log an XML-ready `seatN.position` line. This changes the same transform
  the attach pipeline set from XML, so the logged Y can be baked back into
  `vehicles.xml` and the temporary key bindings removed.
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

## Suspension & ground clearance (WheelCollider tuning)

The engine **never touches** the prefab's `suspensionSpring` /
`suspensionDistance` — whatever the bundle ships is what the car rides
on. (Friction stiffness IS recomputed every tick in `SetWheelsForces`
from `wheel.forwardStiffnessBase` / `sideStiffnessBase`, which are
captured from the prefab at `SetupWheels` — write the bases, not the
collider, if changing grip at runtime.)

### Do the sag math or your car rides on its bump stops

Static sag past the spring's target point = per-corner load / spring
rate. A 1550 kg car needs each corner to carry ~3.8 kN; on 50 kN/m
springs that's 7.6 cm of sag — with 0.16 m travel and `targetPosition`
0.5 the suspension rests ~97% compressed. Symptoms: the car sits
visibly lower than the authored mesh stance, and every bump slams the
chassis collider into terrain because there's no compression travel
left.

Design procedure (hub positions in body frame, pivot at local `P`):

- hub range: `P` (fully compressed) … `P − suspensionDistance` (full droop)
- target hub = `P − suspensionDistance × (1 − targetPosition)`
- equilibrium hub = target + sag, where `sag = load / spring`
- bump travel from rest = `suspensionDistance × (1 − targetPosition) − sag`
  — keep this ≥ ~0.08 m for voxel terrain
- place the pivot so the equilibrium hub sits at (or slightly below)
  the mesh's authored wheel center; putting it a few cm below raises
  the resting stance for clearance.

Visuals stay correct automatically: when `tireSuspensionPercent` > 0,
`EntityVehicle.Update` writes the visual tire transform's local Y from
`WheelCollider.GetWorldPose` every frame, so wheels stay planted at any
ride height.

### Raised physics, stock looks: offset the paint root, not the model

Raising the resting stance for clearance makes the body visibly float
("lifted 4x4" look). Fix it cosmetically without giving up clearance by
dropping the *body visual* back down inside the prefab:

- Do **not** offset `ModelTransform` ("GameObject") — the engine stomps
  its world pose every frame (`SetPosition`, `OriginChanged`,
  `updateTransform` all write `ModelTransform.position`).
- Instead set a negative local Y on the paint root **`M`** (child of
  `ModelTransform`). Nothing in the engine ever writes `M`'s transform,
  so the offset survives; wheel visuals are siblings of `M` and keep
  re-planting from `GetWorldPose`, so tires stay on the ground while
  only the body/interior sink.
- Amount = equilibrium-hub height above the authored wheel center
  (pivot raise − sag-below-target; see the design procedure above).
- **Seats must follow**: seat `position` is applied to the *rider's*
  ModelTransform relative to the vehicle **entity root**, not the body
  mesh — drop seat Y by the same amount or the driver floats above the
  interior. Rider-relative IK hand targets follow the seat, so they
  stay on the (also-dropped) steering wheel.

Worked example: Supra (`SetupSupraPrefab.cs`, `VisualLowerY = 0.039`),
2026-07.

### Chassis colliders on voxel terrain: wheels are the ground contact

A single full-length box collider with a realistic ground clearance
(~0.14 m for a sports car) snags on every bump — the corners of the
front/rear overhangs dig into slope transitions. Shape the boxes so
only the WheelColliders normally touch ground:

- **Belly box**: span the wheelbase only (a little past the axles),
  floor at ~0.30 m in body frame.
- **Nose/tail boxes**: cover the overhangs with a higher floor
  (~0.42 m) so bumpers still collide with walls/entities but slopes
  pass underneath (~30° approach/departure).
- Keep a cabin box for the greenhouse as usual.

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
// V2.x API (post 2026-06-29 update) — id-based commands, void return.
public override void OnEntityActivated(EntityActivationCommand _command, EntityPlayerLocal _playerFocusing)
{
    if (CommandIs(_command.commandId, "drive") || CommandIs(_command.commandId, "ride")) {
        EnterVehicle(_playerFocusing);   // hand off — the pipeline does the rest
        return;
    }
    base.OnEntityActivated(_command, _playerFocusing);   // storage, take, etc.
}

// Scripted-flight vehicles have no engine parts, so base's isDriveable()
// check disables "drive" — override the allow hook to enable it yourself:
public override bool AllowActivationCommand(System.ReadOnlySpan<char> _commandName, EntityPlayerLocal _playerFocusing)
{
    if (CommandIs(_commandName, "drive") || CommandIs(_commandName, "ride"))
        return !IsDead() && /* your seat-availability logic */;
    return base.AllowActivationCommand(_commandName, _playerFocusing);
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
// V2.x API. Do NOT name the stub "Physics" — Entity.PhysicsInit destroys a
// RootTransform child with that exact name once InitCommon has tagged your
// pre-seeded PhysicsTransform with the "Physics" tag (see API-break section).
public override void Init(int _entityClass, EntityInstanceAssets _assets, EModelInstanceAssets _eModelAssets)
{
    if (PhysicsTransform == null && RootTransform != null)
    {
        var physGO = new GameObject("MyVehiclePhysics");
        physGO.transform.SetParent(RootTransform, worldPositionStays: false);
        var rb = physGO.AddComponent<Rigidbody>();
        rb.isKinematic = true;
        rb.useGravity = false;
        rb.mass = 150f;
        PhysicsTransform = physGO.transform;
    }
    base.Init(_entityClass, _assets, _eModelAssets);
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

## Honking the horn as a mod trigger

The horn is a clean, always-available hook for "the driver did something
deliberate while seated" — no new keybind needed. `EntityVehicle.UseHorn(
EntityPlayerLocal player)` is the single call site: `PlayerMoveController.
Update` routes `playerActionsLocal.VehicleActions.HonkHorn.WasPressed`
straight to it while the local player is attached to a vehicle (default
bind **X** / LeftBumper). It plays `hornSound` and fires the vehicle's
`onHonkEvent`, nothing else — so a Harmony **postfix** is a safe place to
bolt on behavior:

```csharp
[HarmonyPatch(typeof(EntityVehicle), nameof(EntityVehicle.UseHorn))]
public static class Patch_EntityVehicle_UseHorn
{
    public static void Postfix(EntityVehicle __instance, EntityPlayerLocal player)
    {
        if (player == null || player.AttachedToEntity != __instance) return;
        // __instance.boundingBox is the vehicle's WORLD-space AABB.
        // ... do your thing, e.g. open a window.
    }
}
```

- Only the **local driver** reaches `UseHorn` (it's called from the local
  `PlayerMoveController`), and `player.AttachedToEntity == __instance`
  confirms they're actually seated in that vehicle, not a passenger of
  another.
- `Entity.boundingBox` (public field) is the world-space AABB — use it for
  "is the vehicle inside volume X" tests without touching colliders.

### Opening a cursor window from the horn = free input lockout

If your horn handler opens an XUi window that carries `cursor_area="true"`
(or you `Open(group, _bModal:true)`), the game **stops feeding the vehicle
any input for as long as the window is up**. `PlayerMoveController.Update`
early-returns on:

```csharp
bool flag = !windowManager.IsCursorWindowOpen()
         && !windowManager.IsModalWindowOpen()
         && (playerActionsLocal.Enabled || playerActionsLocal.VehicleActions.Enabled);
// ...later: if (!(bCanControlOverride && flag) && ...) { stopMoving(); return; }
```

So the moment the window opens: the hardware cursor appears, steering /
throttle / horn all freeze (`stopMoving()` zeroes `movementInput` each
frame), and `UseHorn` can't re-fire to reopen the window. Close the window
and the next frame driving resumes. This is what makes "honk → click a
menu with the mouse, still seated" work without any manual
`Cursor.lockState` juggling or input-suppression bookkeeping. The vehicle
rigidbody physics keeps running while the window is open (only *input* is
gated), so a moving platform under the car still carries it.

Worked example: Elevator mod — honk while a vehicle is fully pulled into an
elevator car pops a modal floor picker (`ElevatorHorn.cs` +
`ElevatorVehicleFloorController`), 2026-07.

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

## Custom paint picker + disabling vanilla cosmetics

Pattern for a per-vehicle "Paint" color wheel replacing the vanilla dye
slot (worked examples: Supra `SupraPaint*`, Oppressor `OppressorPaint*`,
AssettoCar `AssettoPaint` — see the Assetto Corsa Import note):

- **Radial command**: in the entity class, add
  `_addCallback(new EntityActivationCommand("mymod_paint", "mymod_paint"))`
  in `InitLocalActivationCommands`; allow it in `AllowActivationCommand`
  (`!IsDead()`); open the dialog in `OnEntityActivated`. The icon string
  resolves to UIAtlas sprite `ui_game_symbol_<icon>` (ship a PNG in
  `UIAtlases/UIAtlas/`), the label comes from Localization key
  `entitycommand_<commandId>`.
- **Dialog**: IMGUI `MonoBehaviour` singleton (`DontDestroyOnLoad`) with
  an HSV wheel texture, brightness slider, hex field. While open:
  `player.SetControllable(false)`, force cursor visible/unlocked every
  `Update`, `Input.ResetInputAxes()`, ESC closes. Persist on close.
- **Persistence**: colors keyed by `entityId` in a JSON file inside
  `GameIO.GetSaveGameDir()` — entity ids are only meaningful per save,
  and the save folder survives mod redeploys (the deployed mod folder
  does not).
- **Applying color**: depends on the model. Supra tints the chassis
  paint material (`Vehicle.mainEmissiveMat`) after neutralizing its baked
  albedo to white; the Oppressor bakes the color into masked albedo
  pixels (its `ApplyDyeTint` blue-mask pass). If the apply is a full
  CPU texture rebake, throttle the poll (~0.2 s) so wheel-dragging
  previews don't rebuild textures every frame.
- **Disable vanilla cosmetics** (a dye would fight the custom paint):
  1. omit `canHaveCosmetic` from the placeable item's `Tags`;
  2. hide the modify-window dye row with a Harmony postfix on the
     `XUiC_VehicleWindowGroup.CurrentVehicleEntity` setter:
     `__instance.cosmeticGrid.ViewComponent.IsVisible = !(value is MyEntity);`
- If the paint material's `_Color` is the carrier (Supra-style), also
  postfix `Vehicle.SetColors` to re-assert the saved color — the engine
  stomps `_Color` white on spawn/load and on every seat enter/exit.
  Texture-baked approaches (Oppressor) don't need this.

---

## Placement — spawning a vehicle from the placeable item

Vanilla placement (`ItemActionSpawnVehicle.CalcSpawnPosition`) never
inspects the voxel grid. If you write a custom spawn action, mirror it —
**do not** gate placement on `world.GetBlock(...)` solidity tests:

- `Block.shape.IsSolidSpace` is **false for smooth-terrain voxels**, so a
  "hit block must be solid" check rejects ordinary ground (dirt, roads,
  stone) while accepting oddballs like grass decoration blocks. Symptom:
  "can only place the vehicle on grass".
- Vanilla instead raycasts and uses the raw physics hit point, which
  works on smooth terrain, slopes, and block tops alike:

```csharp
// layer mask 8454144 (terrain + chunk colliders), hit mask 69
if (!Voxel.Raycast(world, player.GetLookRay(), dist + 1.5f, 8454144, 69, 0f))
    return false;
Vector3 hitPos = Voxel.voxelRayHitInfo.hit.pos;
```

- Validity = hit point in range, `world.CanPlaceBlockAt(new Vector3i(hitPos),
  world.GetGameManager().GetPersistentLocalPlayer())` (land claim /
  trader protection), and a clearance scan: sweep `Physics.Raycast` rays
  through the vehicle's bounding box (`VehicleSize` from the item XML,
  e.g. motorcycle `1.3, 1.9, 3`) against collider mask `28901376`, in
  Origin-local space. Vanilla retries the box at vertical offsets
  `0.14 … 1.14` in `0.25` steps so uneven ground still finds room.
- Vanilla spawns the entity at `hitPos + 0.25f * Vector3.up` and lets
  physics settle it; a kinematic/scripted vehicle should instead spawn
  (and preview) at its **settled** height. Watch the off-by-one: if the
  vehicle's soft floor is `World.GetHeight(x,z) + hoverOffset`, the
  settled pivot is `surface + (hoverOffset - 1)` because `GetHeight`
  returns the block *below* the walkable surface — but the raycast
  `hit.pos` here IS the surface. Adding the full `hoverOffset` to
  `hit.pos` previews/spawns the vehicle a whole block higher than it
  settles (the Oppressor uses `RestingHoverOffset - 1f`).

Worked example: `SpawnOppressor.TryComputePlacement` (Oppressor mod),
fixed 2026-07 after the voxel-solidity version made normal terrain
unplaceable.

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

**Voxel-only floors sink through runtime platforms** (verified: Oppressor
on the Elevator mod's gliding car, 2026-07). Runtime-moved geometry exists
only as PhysX colliders; the heightmap says the floor is gone, so a
voxel-floored vehicle descends through the platform, then snaps up when the
blocks are re-placed. Fix: max the voxel floor with a downward SphereCast on
the chunk-collider layer (16); skip `distance == 0` hits (initial overlap —
see [Unity Sweep Casts & Resting Contact](Unity%20Sweep%20Casts%20%26%20Resting%20Contact.md)):

```csharp
float floor = world.GetHeight(Mathf.FloorToInt(p.x), Mathf.FloorToInt(p.z)) + hoverOffset;
Vector3 origin = p - Origin.position + Vector3.up * 0.5f;
if (Physics.SphereCast(origin, 0.4f, Vector3.down, out var hit, 8f, 1 << 16)
    && hit.distance > 0f)
    floor = Mathf.Max(floor, hit.point.y + Origin.position.y + hoverOffset - 1f);
```

**Mind the one-block offset between the two sources**: `Chunk.GetHeight`
stores the Y of the *topmost solid block* (see `Chunk.RecalcHeights` —
`m_HeightMap[i] = y` of the first non-air block), which is one below the
walkable surface, while `hit.point.y` is the *actual surface*. If both get
the same `+ hoverOffset`, the collider floor sits exactly one block higher
and wins the `Max()` — and since **terrain chunk colliders are on layer 16
too** (the game's own roof-check raycast in `EntityVehicle.OnEntityActivated`
uses mask `65536`), it wins *everywhere*, pinning the vehicle a full block
off the ground. Subtract `1f` on the collider branch (verified with
Oppressor, 2026-07-09, game buildid 23906531).

The downward cast only ever finds floors at/below the vehicle, so normal
terrain behavior (incl. the `GetHeight` whole-column/roof quirk) is
unchanged. See EntityOppressor.FloorYAt and
[Moving Platforms & Riding Entities](Moving%20Platforms%20%26%20Riding%20Entities.md).

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
<<<<<<< Updated upstream
| Windows/glass render opaque white (or dye colors the wrong part) | `VehiclePart.SetColors` stomped material slot 0 of the first renderer under the `paint` transform with the tint (white when undyed). Put the car-paint material at slot 0 and author its color in the albedo texture with white `_Color` — see the paint/dye contract above |
=======
| Placeable item only places on grass / odd blocks, not normal ground | Custom spawn action gates on `Block.shape.IsSolidSpace`, which smooth terrain fails. Use the vanilla raycast + physics hit point instead (see **Placement**) |
>>>>>>> Stashed changes

---

## Related

- [Entities](Entities.md) — base entity class, AI tasks, drops
- [Asset Bundles](Asset%20Bundles.md) — loading the bike prefab
- [XML Patching (XPath)](XML%20Patching%20(XPath).md) — how to `<append>` to `vehicles.xml`
