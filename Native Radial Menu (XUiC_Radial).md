# Native Radial Menu (XUiC_Radial)

Part of the [7DTD Modding Knowledgebase](README.md). How to drive the game's built-in radial wheel (the vehicle/turret/junk-drone/quick-menu wheel) from a mod — no custom XML or controller needed.

---

## TL;DR

The radial wheel is the vanilla `windowRadial` window driven by the `XUiC_Radial` controller. A mod can populate and open it directly through the public API; **you do not need to ship any XUi XML or register a controller.**

```csharp
using GUI_2; // for UIUtils.ButtonIcon

LocalPlayerUI ui = LocalPlayerUI.GetUIForPrimaryPlayer();
XUiC_Radial radial = ui.xui.RadialWindow;        // persistent controller, ticks even while closed

radial.Open();                                    // vanilla calls Open() FIRST, then populates
radial.ResetRadialEntries();
radial.CreateRadialEntry(0, "ui_game_symbol_drone", "UIAtlas", "", "My selection text");
radial.CreateRadialEntry(1, "myCustomSprite", "ItemIconAtlas", "", "Other");
radial.SetCommonData(UIUtils.ButtonIcon.RightStick, OnRadialCommand);

void OnRadialCommand(XUiC_Radial r, int commandIndex, XUiC_Radial.RadialContextAbs ctx)
{
    // commandIndex is the int you passed to CreateRadialEntry for the picked wedge
}
```

`CreateRadialEntry(int commandIdx, string iconSprite, string atlas = "UIAtlas", string text = "", string selectionText = "", bool highlighted = false)` — there's also an overload taking a `Color` iconTint after the sprite. `selectionText` is shown in the wheel's centre label while that wedge is hovered.

`SetCommonData(UIUtils.ButtonIcon selectButtonIcon, CommandHandlerDelegate handler, RadialContextAbs context = null, int preSelectedCommandIndex = -1, bool hasSpecialActionPriorToRadialVisibility = false, RadialStillValidDelegate validityCallback = null)`.

Key access paths (all public in the publicized `Assembly-CSharp`):
- `LocalPlayerUI.GetUIForPrimaryPlayer().xui` → `XUi`
- `XUi.RadialWindow` → `XUiC_Radial` (set in `XUiC_Radial.Init` via `xui.RadialWindow = this`)
- `UIUtils.ButtonIcon` lives in namespace **`GUI_2`** — `RightStick` is a safe value for the select-button callout.
- `RadialContextAbs` and `CommandHandlerDelegate` are nested in `XUiC_Radial` (`XUiC_Radial.RadialContextAbs`, `XUiC_Radial.CommandHandlerDelegate`).

## How it drives itself (you don't manage the window)

After `Open()`, `XUiC_Radial.Update()` (which runs even while the window is closed) does everything:
- waits `displayDelay` (0.25 s), then opens `windowRadial` modal (`windowManager.Open(WindowGroup, _bModal:true, _bIsNotEscClosable:true)`);
- keeps the wheel open while **any** of these actions is held — `Activate`, **`Reload`**, `ToggleFlashlight`, `Inventory`, `Swap`, `InventorySlotLeft/Right`, `QuickMenu` (see `radialButtonPressed`);
- steers the highlight by stick/mouse direction (`Look.Value`, falling back to `Move.Value`) via `CalculateSelectionFromController`, plus per-entry mouse `OnHover`;
- on release, fires `handler(radial, commandIndex, context)` then closes.

**"Hold a key, release to select" comes for free** — just open the wheel while that key is held. Because `Reload` (default **R**) is in the keep-open list, opening on R-down gives a hold-R wheel that selects on R-up. (For a held-item context that vanilla doesn't already give a radial, trigger the open yourself from a `MonoBehaviour.Update` that detects the key + the held item.)

## Icons

- Vanilla UI symbols are `ui_game_symbol_*` in the `UIAtlas` atlas (e.g. `ui_game_symbol_hammer`, `ui_game_symbol_drone`, `ui_game_symbol_map`, `ui_game_symbol_chat`). There is **no** vanilla gear/settings/cog symbol.
- Custom sprite: drop a white-on-transparent PNG in `UIAtlases/ItemIconAtlas/<name>.png` (auto-discovered by filename, like item icons) and reference it with `CreateRadialEntry(idx, "<name>", "ItemIconAtlas", …)`. `UIAtlases/UIAtlas/` also works for the `UIAtlas` atlas. A new sprite needs a deploy + restart (not just `xui reload`) to rebuild the atlas.

## Held-item radial (native hook, alternative)

`PlayerMoveController` calls `holdingPrimary.SetupRadial(playerUI.xui.RadialWindow, entityPlayerLocal)` when the held item reports a radial — so a custom `ItemClass`/holding object can supply wheel entries the "proper" way. But the trigger key is the vanilla radial action, not necessarily R; driving the wheel yourself (above) is simpler when you want a specific key.

## Version gotcha — 2026-06-29 explosion API change

The 2026-06-29 game build changed `GameManager.ExplosionServer`: it **dropped the leading `int clrIdx` argument** (now 8 params: `Vector3 worldPos, Vector3i blockPos, Quaternion rotation, ExplosionData data, int entityId, float delay, bool removeBlockAtExpl, ItemValue source = null`). Correspondingly `HitInfoDetails.clrIdx` was removed. Mods calling the old 9-arg overload throw `MissingMethodException` at detonation after updating. Drop the `clrIdx` arg and stop reading `hit.clrIdx`.
