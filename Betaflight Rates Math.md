# Betaflight Rates Math

Reference for simulating a Betaflight rateprofile (stick → deg/s, throttle
shaping) in a mod flight model. Implemented in `FPV/src/FPVRates.cs`; edited
live on the FPV Options dialog's **Rates** tab (`FPVOptionsDialog.DrawRatesTab`).

## Rates curve (Betaflight rates type, from `fc/rc.c applyBetaflightRates`)

Inputs per axis, configurator conventions: **RC Rate** (~1.0, CLI stores ×100),
**Rate / super rate** (0..0.99), **RC Expo** (0..1). Stick `s` in -1..1,
`a = |s|` **before** expo (the superfactor uses the pre-expo magnitude — easy
to get wrong).

```
if expo > 0:   s = s·a³·expo + s·(1 − expo)          # bends the centre soft
rcRate = RC Rate;  if rcRate > 2: rcRate += 14.54·(rcRate − 2)   # the >2.0 kink
rate  = 200 · rcRate · s
if super > 0:  rate /= clamp(1 − a·super, 0.01, 1)   # steepens toward full stick
```

- Expo does not change the full-stick value (at a=1 the blend is identity), so
  **max velocity = 200 · rcRate / (1 − super)** — the configurator's "Max Vel"
  column. Sanity checks: 1.0/0.7 → 667°/s, 1.2/0.7 → 800°/s, 1.5/0.65 → 857°/s.

## Throttle curve (MID/EXPO, from the firmware throttle lookup, normalized)

Mid `m`, expo `e`, stick `t` in 0..1:

```
tmp = t − m
half = tmp ≥ 0 ? 1 − m : m           # width of the half being traversed
out = m + tmp · (1 − e + e·tmp²/half²)
```

Exact at the ends (0→0, 1→1), pivots at `m`; `e` flattens response around the
pivot. **Throttle limit is applied after the curve** (it lives in the mixer):
`SCALE` multiplies by limit%, `CLIP` is `min(out, limit%)`.

## TPA (throttle PID attenuation)

Breakpoint is in RC µs (1000..2000). Normalized: `bp = (breakpoint − 1000)/1000`,
factor `= 1 − tpa · clamp01((thr − bp)/(1 − bp))`, computed from the
**post-curve** throttle. On real firmware TPA trims PID gains; a sim without a
PID loop applies the factor to the commanded rates instead — same knob, same
direction (more TPA = softer response at high throttle).

## UI notes (IMGUI)

- Curve previews: rasterize into a cached `Texture2D` and rebuild only when a
  plotted value changes (hash the inputs); per-frame redraws with hundreds of
  `GUI.DrawTexture` calls are needlessly hot. `Texture2D` y=0 is the bottom row
  and `GUI.DrawTexture` keeps it there, so plot y-up directly.
- Slider drags: write the config object live (flight picks it up next frame)
  but defer the JSON `Save()` to mouse-up, or a drag writes the file at 60 Hz.

## Building FPV outside deploy.ps1

`FPV/src/FPV.csproj` defaults `GameDir` to `C:\Program Files (x86)\Steam\...`,
which is wrong on this machine — the game is at
`D:\SteamLibrary\steamapps\common\7 Days To Die`. `deploy.ps1` discovers the
real path from the Steam registry/libraryfolders and passes it; a direct
compile check needs `dotnet build src/FPV.csproj -p:GameDir="D:\SteamLibrary\steamapps\common\7 Days To Die"`.
