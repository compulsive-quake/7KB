# HUD Safe Zones for Custom Windows & Overlays

When a custom window or IMGUI overlay is shown **without** closing the game's HUD, two HUD regions remain drawn on top of the 3D world and must not be overlapped:

1. **Top strip — compass + `DAY: N  TIME: hh:mm  temp°F` line**
   The compass sits at the very top, with the DAY/TIME/temperature line directly beneath it. The two together occupy roughly the first 110 screen pixels (≈55 scaled GUI units at 2× scale). Always visible outside of fullscreen menus.

2. **Bottom strip — message line + toolbelt**
   The yellow/green message text (e.g. *"THE DUKE LEFT YOU A NOTE TO READ"*) and the 10-slot toolbelt sit together at the bottom center.

Do not draw over either strip unless your UI explicitly suppresses them (e.g. by opening through `GUIWindowManager` with an exclusive window group that hides the HUD, like the Map or Main Menu).

## Rule

For IMGUI / OnGUI overlays and free-floating panels, clamp to the vertical range:

```
topSafe    ≈ 60 scaled GUI units   (below compass + DAY/TIME row)
bottomSafe ≈ 80 scaled GUI units   (above message line + toolbelt)
```

Reference implementation (see `RetroZed/src/RomBrowserController.cs`):

```csharp
const float topSafe = 60f;
const float bottomSafe = 80f;
float safeH = Screen.height / scale - topSafe - bottomSafe;
float py = topSafe + Mathf.Max(0f, (safeH - panelH) * 0.5f);
```

This centers the panel inside the safe zone rather than naively centering to screen height.

## Polling "is the compass / gameplay HUD visible right now"

For an overlay that should appear only during normal play (like the vanilla
compass — hidden inside the map, crafting, backpack, trader, etc.), poll two
things on `LocalPlayerUI.GetUIForPrimaryPlayer().windowManager`:

```csharp
bool hudVisible = wm.IsHUDEnabled()            // HudEnabledStates.Enabled (not Partial/Full hide)
               && wm.IsWindowOpen("compass");   // the "compass" *window group* id
```

Why this is the right signal (verified by decompiling the 2026-07 build):

- The vanilla `windowCompass` lives in the `compass` window group
  (`xui.xml`), and every fullscreen menu group carries
  `close_compass_on_open="true"` — `XUiWindowGroup.OnOpen` calls
  `windowManager.Close("compass")`, so `IsWindowOpen("compass")` goes false
  exactly when a menu is up. The id passed is the **group** name `"compass"`
  (registered via `windowManager.Add(this)` in `XUiWindowGroup.Init`), not a
  window name.
- `XUiC_CompassWindow.Update` itself gates on
  `!localPlayer.IsDead() && windowManager.IsHUDEnabled()`; `IsHUDEnabled()`
  returns `bHUDEnabled == HudEnabledStates.Enabled` (enum:
  `Enabled/PartialHide/FullHide`), covering the player's HUD-hide toggle.

An IMGUI overlay (e.g. Uncover's minimap) can poll this from `MonoBehaviour`
without registering any XUi window. A native XUi window gets the same behavior
for free by being appended to the `compass` group and copying that `IsVisible`
line into its controller `Update`.

## When the rule does not apply

If your UI opens via the XUi window system with a group that sets `HUDDisabled` / replaces the HUD (e.g. the backpack, crafting, map, escape menu), the HUD strips are hidden while the window is open — screen-centering is fine there.

For IMGUI panels drawn via a `MonoBehaviour.OnGUI` (e.g. alignment tools, settings panels, overlays that co-exist with gameplay), the HUD strips stay drawn and the safe-zone rule applies.

## Making the HUD dodge YOUR overlay instead

When a persistent overlay must live where HUD windows already sit (e.g. the Supra tachometer in the bottom-right), invert the rule: lift the HUD windows out of the way while the overlay shows. Recipe — generic XUi overlap-lift (`xui.windows` + `xui.GetXUiWindowScreenBounds` + `xui.GetPixelRatioFactor`, offset `view.Position`, restore on hide) plus the `Uncover.MinimapSettings.ExternalBottomReservePx` reserve handshake for IMGUI widgets — documented in [AC D Tachometer Port (IMGUI gauge HUD)](<AC D Tachometer Port (IMGUI gauge HUD).md>). Key rule: one writer per window Position — coordinate via a published reserve, never two self-healing writers.
