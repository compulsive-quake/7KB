# zPhone External Apps

zPhone can expose lightweight app pages from sibling mods without requiring those mods to patch zPhone directly. Add a `zphone/app.json` folder at the mod root. The zPhone registry scans sibling mod folders for this manifest and renders the referenced JSON UI page.

Minimal manifest:

```json
{
  "schema": 1,
  "id": "rocket-turret",
  "label": "Rocket Turret",
  "page": "rocket-turret",
  "ui": "zphone/page.json",
  "assembly": "RocketTurret",
  "type": "RocketTurretZPhoneActions",
  "order": 115
}
```

If `icon` is omitted, zPhone falls back to its settings icon. Keep the `zphone/` folder deployable with the mod; ModForge deploy filters should not exclude it.

The manifest `type` should be a public class in the named mod assembly with a public static `Register()` method. In that method, register actions by reflecting the loaded `zPhone` assembly and calling `ZphoneBridge.RegisterAction(string, Action<JObject>)`. This avoids a compile-time zPhone reference. The same bridge exposes `SendToJs(string, object)` for status/state updates.

**Symptom: app stuck on black "Loading" screen.** The page's `stateAction` (e.g. `rocketTurret.requestState`) fires on open and the page waits for the matching `stateEvent` (e.g. `rocketTurret.state`) before rendering. If the mod ships `zphone/app.json` + `page.json` but no actions class (or the class lacks a public static `Register()`), `ZPhoneAppRegistry.RegisterExternalActions()` logs `App '<id>' did not expose <Type>.Register()`, no handler answers the state request, and the page hangs on "Loading" forever. Fix: add the `<Type>` class with `Register()` wiring `*.requestState`/`*.setFloat`/`*.action` and a `SendState()` that emits the `stateEvent` with `installed = true` plus every `stateField` the page references. RocketTurret's class mirrors FPV's (minus prop-spin) — copy that pattern. The class lives in the mod DLL, so this is a restart-kind change (deploy + game restart, not xui_reload).

External page JSON supports controls such as `button`, `number`, `toggle`, `select`, `heading`, `text`, `stat`, and `info`. A common tuning layout uses:

- `stateAction` / `stateEvent` for initial state and refreshes.
- Number controls that send `{ field, value }` through a `*.setFloat` action.
- Button controls that send `{ method }` through a `*.action` action.

For held-item tuning apps, persist the chosen values under the mod's `Config/` folder, load them from `IModApi.InitMod`, and apply live transforms in a `MonoBehaviour.LateUpdate()` after the game has positioned the held model. HoldType can be updated at runtime by reflecting the target `ItemClass`, setting a `HoldType` field/property when present, and updating its `DynamicProperties.Values["HoldType"]` string as a fallback. Refresh the currently held item with `inventory.SetRightHandAsModel()` and `player.ShowHoldingItem(true)` after HoldType changes.

Tuning apps are scaffolding: once values are finalized, promote them to hardcoded defaults / `items.xml` and delete the app (`zphone/` folder, the actions class, the settings JSON load/save). **RocketTurret did this in June 2026** for hand placement — that tuning now lives hardcoded in `RocketTurretPointerHandAdjuster` (see Items.md "Held Item Scale & Positioning" for the end-state pattern). Later in June 2026 RocketTurret briefly re-created the app for a different tunable (laser beam origin offset + width — `RocketTurretZPhoneActions` + `RocketTurretLaserTuning`, saving `Config/rocket_turret_laser.json`), then **deleted it again** the same month when the whole target-painter was reworked: the custom pointer model was dropped for the vanilla flashlight and the laser logic was replaced wholesale with the Airstrike designator's (which carries its own **numpad** live-tuner, `AirstrikeLaserOrigin`/`RocketTurretLaserOrigin`, instead of a zPhone app). Net lesson: if you're porting laser logic from a mod that already has an in-game tuner, you inherit that tuner — don't also stand up a zPhone app for the same values. Then (still June 2026) the app was **re-created a third time** for a different purpose — tuning the turret's procedural **aim** (`RocketTurretZPhoneActions` + `RocketTurretAimTuning`, saving `Config/rocket_turret_aim.json`). It exposes two control sets because the model has two driven bones (a yaw "top half" that spins to face the target and a "head" that leans back for elevation); each set is a `select` choosing which **local axis** (X/Y/Z) the bone rotates around plus an invert `toggle`, with a `toggle` for a red launch-direction debug line. This is the canonical use of `select`: options `[{value:0,label:"X"},…]`, dispatched through a `*.setFloat` action as `{field, value}` (the webapp's `SelectRow` sends the chosen `value` as a number), handled by the same `SetFloat(field, value)` switch as numbers. **FPV finalized its hand-placement tuning the same way in June 2026** (hardcoded in `FPVHandAdjuster`, `HoldType` baked into `items.xml`, whole `FPVControlAPI` + Newtonsoft.Json reference deleted with the app).

When deleting an app, also clean the **deployed** mod folder: a `Copy-Item`-based deploy script (unlike `robocopy /MIR`) never removes files, so the stale `zphone/` folder keeps the app visible in zPhone (now broken, since its actions class is gone) and the orphaned settings JSON lingers under `Config/`.

The `toggle` control is a button-style control: `{ "type": "toggle", "label", "action", "method", "stateField" }`. It sends `{ method }` through the action (same shape as `button`) and renders on/off from the boolean `stateField` in the state event — so the C# side implements it as a parameterless method that flips the bool, and `SendState()` reports it.

Note the deploy gotcha for saved tuning files: a `robocopy /MIR` deploy (ModForge default) deletes runtime-saved JSON under the deployed mod's `Config/` on every redeploy, because the file doesn't exist in the source repo. Read the saved values out of the deployed folder (or copy the JSON into the repo's `Config/`) before the next deploy.
