# Player Feedback Sounds (UI / "denied" buzzers)

How to play the stock UI feedback sounds from C# and which clip name maps to
which situation. Useful when a mod gates an action and wants the same audio cue
the vanilla game uses.

## Playing a sound in the player's head

```csharp
using Audio;
// ...
Manager.PlayInsidePlayerHead("missingitemtorepair");
```

`Audio.Manager.PlayInsidePlayerHead(string clipName, int entityId = -1,
float delay = 0, bool isLooping = false, bool isUnique = false)` — the
single-arg overload is the common case for a one-shot 2D UI cue. It is *not*
spatialized; use it for operator/UI feedback, not world sounds. For a 3D world
sound played by an entity, use `entity.PlayOneShot(clipName)` instead.

## Known clip names

| Clip | When vanilla plays it |
| --- | --- |
| `missingitemtorepair` | Trying to **upgrade/repair a block without the required materials** (e.g. swinging a stone axe at an upgradeable block you can't afford). Routed through `XUiC_CollectedItemList.AddItemStack` whenever an `ItemStack` with `count == 0` is pushed via `EntityPlayerLocal.AddUIHarvestingItem(stack, true)`. This is the classic "can't do it" buzzer. |
| `twitch_no_attack` | Item/attack disabled (e.g. `PassiveEffects.DisableItem` active). |
| `player_death_stinger`, `spawnInStinger`, `ui_weather_alert` | Death / spawn / weather-alert stingers (see `EntityPlayerLocal`). |

## How these were found

Decompile `Assembly-CSharp.dll` with `ilspycmd` and grep for
`PlayInsidePlayerHead` / `PlayOneShot`:

```
ilspycmd.exe -t ItemActionRepair "<game>/7DaysToDie_Data/Managed/Assembly-CSharp.dll"
ilspycmd.exe -t XUiC_CollectedItemList "<...>/Assembly-CSharp.dll"
```

`ilspycmd` is installed as a global dotnet tool (`~/.dotnet/tools/ilspycmd.exe`)
and `-t <TypeName>` decompiles a single class — fast way to confirm exact sound
strings and call sites instead of guessing.
