# Time & Weather Control

How to freeze/set the world clock and force weather from a mod (verified against 7DTD 3.0 decompile, July 2026; used by zPhone's God-app "Clear Noon" toggle).

## Freezing / advancing time

- The clock is advanced by `GameManager.updateTimeOfDay()`, which reads `GameStats.GetInt(EnumGameStats.TimeOfDayIncPerSec)`. **Setting it to 0 freezes time-of-day** (vanilla still calls `world.SetTime(world.worldTime)` every 100 ms to keep clients synced).
- Default value is derived in `GameModeAbstract.Init`: `TimeOfDayIncPerSec = 24000 / (DayNightLength_minutes * 60)`. Restore a frozen clock by recomputing this from `GamePrefs.GetInt(EnumGamePrefs.DayNightLength)` — don't cache the old value.
- **It is re-derived at every world start**, so a persistent freeze must be re-applied after world load (zPhone uses a once-per-second enforcer MonoBehaviour).
- Set an absolute time with `world.SetTime(day * 24000UL + hour * 1000UL)`. One game day = 24000 ticks, noon = 12000.

## Forcing weather

Static fields on `WeatherManager`; sentinel values mean "not forced":

| Field | Off value | Notes |
|---|---|---|
| `forceClouds` / `forceRain` / `forceSnowfall` / `forceWind` | `-1f` | 0..1 range |
| `forceTemperature` | `-100f` | °F. Rain falls as snow at ≤ 32 |

- `WeatherManager.BaseTemperature` is a public const = **70** (°F) — the "default forest" baseline temperature.
- After changing force fields set `WeatherManager.needToReUpdateWeatherSpectrums = true` and call `WeatherManager.Instance.CloudsFrameUpdateNow()` so the sky updates immediately.
- **`WeatherManager.Cleanup()` resets all force fields on world exit** — same re-apply caveat as the time freeze.

## Blocking status effects (buffs) on the player

All status effects (bleeding, infection, broken limbs, stun, food/water debuffs) route through `EntityBuffs.AddBuff(string, ...)`. A Harmony prefix returning `false` when `__instance.parent is EntityPlayerLocal` blocks them at the source — removing after the fact via `RemoveBuff` does NOT undo the buff's `onAdded` triggers, so block in the prefix. Resolve the method with a reflection `TargetMethod()` (first `AddBuff` overload whose first param is `string`) because TFP has shipped several signature variants — see the pattern in "Harmony Patching". Implementations: zPhone `GodNoStatusChange.cs`, Scepters `BuffPatch.cs`.
