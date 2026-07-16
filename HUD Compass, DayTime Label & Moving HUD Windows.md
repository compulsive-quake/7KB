# HUD Compass, Day/Time Label & Moving HUD Windows

Part of the [7DTD Modding Knowledgebase](README.md). Verified against the July
2026 Steam build (Uncover mod HUD-layout options).

## Hiding the compass — the vanilla switch

`XUiC_CompassWindow.ShowCompass` is a **public static bool** the game already
honors: the compass sprites/texture in `windowCompass` bind their visibility to
`{show_compass}`, and the DAY/TIME/temperature label exists twice in the XML —
one spot under the compass (`{showtimecompass}`) and one at the very top edge
(`{showtimenocompass}`) — so with `ShowCompass = false` the strip disappears
and the label relocates to the top automatically. No Harmony needed; just set
the static (re-assert it each frame if other code might flip it).

## The DAY/TIME/temp label internals (windowCompass bindings)

- Values: `GameUtils.WorldTimeToElements(world.worldTime)` (day/hour/minute
  tuple), titles `Localization.Get("xuiDay"/"xuiTime")`, temperature
  `XUiM_Player.GetOutsideTemp(player)` gated by `World.TemperatureSurvival`.
- Display gates: `EffectManager.GetValue(PassiveEffects.NoTimeDisplay, ...) == 0`
  and `SandboxOptions.SandboxOptionManager.GetBool(SandboxOptions.ShowDayTime)`
  (note: `SandboxOptionManager` lives in the `SandboxOptions` NAMESPACE next to
  an enum of the same name — fully qualify or `using SandboxOptions;`).
- Day turns red when `World.BloodMoonWarningHour != -1 &&
  GameStats.GetInt(EnumGameStats.BloodMoonDay) == day && warnHour <= hour`.
- Temp colors: ≤32 `0099FF`, ≤50 `00FFFF`, ≥100 `FF0000`, ≥85 `FF8000`.
- To suppress just the label (e.g. to redraw it elsewhere), Harmony-postfix
  `XUiC_CompassWindow.GetBindingValueInternal(ref string value, string bindingName)`
  and overwrite `value = "False"` for `showtimecompass` / `showtimenocompass`.

## Moving vanilla HUD windows at runtime

`XUiView.Position` (`Vector2i`, the XML `pos`) has a working runtime setter
(marks the view dirty; anchor is preserved — pos is relative to it). Recipe:

1. Resolve the window: `xui.GetChildById("windowQuestTracker")` (searches all
   window groups; window ids = `<window name=...>`).
2. Capture `view.Position` as the original once per resolve, then set
   `original + offset` / restore `original`.
3. Re-resolve when the cached controller's `ViewComponent.UiTransform == null`
   (UI rebuilt); re-apply every frame guarded by `if (view.Position != want)` —
   XUi re-parses XML pos on rebuild, so idempotent enforcement self-heals.

Top-right HUD occupants (toolbelt group): `windowLocation` (pos -259,-40),
`windowQuestTracker` / `windowRecipeTracker` (both -255,-88), all anchored
TopRight. Bottom-right: `HUDRightStatBars` (anchored BottomRight at 0,0).
XUi positions are in the 1080p-authored unit space; convert real screen px via
`xui.GetPixelRatioFactor()`. See `HudLayout` / `HudStatBarShift` in Uncover.
