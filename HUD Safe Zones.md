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

## When the rule does not apply

If your UI opens via the XUi window system with a group that sets `HUDDisabled` / replaces the HUD (e.g. the backpack, crafting, map, escape menu), the HUD strips are hidden while the window is open — screen-centering is fine there.

For IMGUI panels drawn via a `MonoBehaviour.OnGUI` (e.g. alignment tools, settings panels, overlays that co-exist with gameplay), the HUD strips stay drawn and the safe-zone rule applies.
