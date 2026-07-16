# AC "D Tachometer" Port (IMGUI gauge HUD)

The Assetto Corsa **D Tachometer v2.2** Python app (Initial D arcade gauges)
ported into 7DTD as an IMGUI HUD. First done in the [Supra](Vehicles.md) mod
(`src/SupraTachometer.cs`, toggle + theme gallery inside `SupraPaintDialog`);
the recipe generalizes to any vehicle mod. Related: [Assetto Corsa Import
(kn5, FSB5, engine audio)](<Assetto Corsa Import (kn5, FSB5, engine audio).md>).

## Theme pack anatomy (per theme folder `themes/D3..D7`)

- `theme_config.ini` — all layout in **background-pixel coordinates**; parse
  case-insensitively (one key is `background_Pedal_height`). Sections:
  `[General]` feature flags, `[Background]`, `[Gear]`, `[RPM Gauge]`,
  `[RPM Bar]`, `[Speed]`, `[Speed Unit]`, `[Pedal]`, `[Drift Light]`,
  `[Rev Light]`.
- `background/background.png` + `labels_{8k,9k,10k,13k,16k,18k,20k}.png` —
  dial numbers are a separate overlay drawn at the `[RPM Gauge]` rect. The
  app picks the variant by the car's maxRpm via a top-down cascade over
  `maxRpm_state_6..0` (falls back to the 10k dial). A 7000-rpm redline lands
  on `labels_8k` in every theme.
- `rpm_bar/rpm_bar.png` — the needle, drawn UNROTATED at `[RPM Bar]`
  rpm_x/y/w/h, then rotated about `rpm_center_x/y` by
  `deg = rpm / (dialMax / degree_available) + degree_offset` (clockwise,
  y-down screen space). dialMax comes from the labels variant (8k → 8000).
- `gears/gear_0..10.png` — **0 = R, 1 = N, 2 = 1st … 10 = 9th** (AC's
  acsys.CS.Gear convention).
- `speed_digits/speed_digits_0..9.png` — digits drawn right-to-left:
  the ones digit sits at `speed_x`, each next digit at
  `speed_x - i*(speed_width + speed_gap)`. D7 instead has
  `speed_red/yellow/blue` sets (`variable_speed_color=True`): red < 100,
  yellow < 150, blue ≥ 150 (thresholds on the displayed number).
- `speed_unit/kmh.png|mph.png`, `preview.png` (360×144, 2.5:1),
  D7 also `rev_light.png`: alpha 1 within 250 rpm of redline, else fades in
  over a 2000-rpm window (`(rpm - maxRpm)/2000 + 1`).
- Night variants (`night_labels_*`, `night_rpm_bar`) exist in D5-D7 — keyed
  to headlights in AC; unported so far.

## 7DTD-side notes

- **IMGUI needle rotation**: don't trust `GUIUtility.RotateAroundPivot`
  composition with a scaled `GUI.matrix` — build the matrix explicitly:
  `GUI.matrix = TRS(pivotScreen, Euler(0,0,deg), 1) * TRS(-pivotScreen, id, 1)
  * scaleMatrix`, draw the needle rect in virtual coords, restore. Positive z
  angle = clockwise in GUI space, which matches the app's math exactly.
- **PNG loading needs `UnityEngine.ImageConversionModule.dll`** referenced in
  the csproj (`ImageConversion.LoadImage`). Load once per theme, destroy the
  old theme's textures on switch (5 themes ≈ 5 MB of PNGs on disk).
- **Gear/RPM are synthesized** (7DTD vehicles have no drivetrain): signed
  forward speed = `Vector3.Dot(ev.vehicleRB.velocity, ev.transform.forward)`;
  an 8-gear band table over top speeds picks the gear (with ~1.5 m/s downshift
  hysteresis), rpm lerps base→redline across the band, smoothed with
  `Mathf.Lerp(rpm, target, 1-exp(-6*dt))`. Reverse gear from negative dot.
- Settings (enabled/theme/unit) persist per save next to `supra_paint.json`
  — save-folder JSON survives redeploys, mod-folder Config/ does not.
- Theme assets ship from a mod-root folder (e.g. `Tachometer/themes/`);
  ModForge classifies them as **assets** drift (no restart needed), but the
  DLL change that renders them is **restart** drift.

## Pushing other HUD elements out of the corner (SupraHudClearance)

The bottom-right corner is contested: vanilla `HUDRightStatBars` (XUi,
anchor BottomRight — vehicle fuel/health, held-item bar, pickup toasts) and
Uncover's IMGUI minimap. Two mechanisms, both in
`Supra/src/SupraHudClearance.cs`:

- **Generic XUi lift** — the game hands you everything needed:
  `xui.windows` is a flat `List<XUiV_Window>`;
  `xui.GetXUiWindowScreenBounds(view.UiTransform)` returns screen-pixel
  (y-up) bounds of a window's sprites; `xui.GetPixelRatioFactor()` converts
  screen px → XUi units (multiply). Any visible window whose bounds overlap
  the reserved rect gets `view.Position = basePos + (0, liftUnits)`;
  restore `basePos` when the overlay hides. Gotchas: compare the *unshifted*
  bounds (subtract your own applied lift) or the fix un-fixes itself next
  scan; windows with no `UIBasicSprite` children report a degenerate
  projected point (skip size < 2 px); skip near-fullscreen bounds (fades /
  vignettes); track views by reference so HUD rebuilds re-resolve; only
  write Position when it drifts (XUi re-parses pos on rebuild —
  self-healing, same trick as Uncover's `HudStatBarShift`).
- **Cross-mod reserve handshake** — never have two mods write the same
  window's Position (they fight at frame rate). Uncover already owns
  `HUDRightStatBars`, so Supra skips it when Uncover is loaded and instead
  sets `Uncover.MinimapSettings.ExternalBottomReservePx` (public static
  float, screen px measured up from the bottom edge) via reflection
  (`Type.GetType("Uncover.MinimapSettings, Uncover")`, cache the
  FieldInfo). Uncover adds the reserve to its minimap's bottom margin and
  into HudStatBarShift's target (converted px→units with
  `GetPixelRatioFactor`), stacking bars above minimap above tachometer.
  Any future bottom-right overlay can publish the same field.
