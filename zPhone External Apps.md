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

External page JSON supports controls such as `button`, `number`, `toggle`, `select`, `heading`, `text`, `stat`, and `info`. A common tuning layout uses:

- `stateAction` / `stateEvent` for initial state and refreshes.
- Number controls that send `{ field, value }` through a `*.setFloat` action.
- Button controls that send `{ method }` through a `*.action` action.

For held-item tuning apps, persist the chosen values under the mod's `Config/` folder, load them from `IModApi.InitMod`, and apply live transforms in a `MonoBehaviour.LateUpdate()` after the game has positioned the held model. HoldType can be updated at runtime by reflecting the target `ItemClass`, setting a `HoldType` field/property when present, and updating its `DynamicProperties.Values["HoldType"]` string as a fallback. Refresh the currently held item with `inventory.SetRightHandAsModel()` and `player.ShowHoldingItem(true)` after HoldType changes.

## Gotcha: leftover deployed manifests

zPhone scans deployed mod folders for `zphone/app.json` — not the source repo. If a mod deletes `zphone/app.json` from source but the deploy script never cleans the destination, the deployed copy survives and zPhone keeps registering the app. When the player opens that orphaned page the bridge logs `no handler for action '<id>.requestState'` (the C# `Register()` is gone). Fixes: either drop the leftover deployed folder, or have the deploy script copy `zphone/` with a pre-clean (remove-then-copy) so source deletions propagate. Symptom in logs: `[zPhone] Bridge: no handler for action 'X.requestState'`.
