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
| `item_pickup` | Generic "picked up an item" cue. `item_plant_pickup` is the crop-harvest variant. Both are `SoundDataNode`s in `Data/Config/sounds.xml`; they are NOT string literals in `Assembly-CSharp.dll` (driven from XML), but you can play them by name. |

## Showing the bottom-right "+N" pickup popup (with sound)

The bottom-right collected-item list is fed by
`EntityPlayerLocal.AddUIHarvestingItem(ItemStack)` →
`XUiC_CollectedItemList.AddItemStack`. It does **not** play a pickup sound on
its own (it only plays `missingitemtorepair` when the stack `count == 0`). Play
the cue yourself. Vanilla crop-harvest pattern (decompiled `BlockPlantGrowing`):

```csharp
var display = stack.Clone();        // clone BEFORE AddItem — see gotcha below
if (player.inventory.AddItem(stack) || player.bag.AddItem(stack))
{
    player.PlayOneShot("item_pickup");   // "item_plant_pickup" for crops
    player.AddUIHarvestingItem(display);
}
```

**Gotcha:** `Inventory.AddItem` / `bag.AddItem` mutate the passed stack's
`count` down to 0 on success. If you then pass that same stack to
`AddUIHarvestingItem`, it shows "0" and fires the `missingitemtorepair` buzzer.
Always clone a display stack *before* adding (vanilla does `itemStack2 =
itemStack.Clone()` for exactly this reason). Used by FPV's drone-return code.

## Shipping & playing your own audio (bypass sounds.xml entirely)

You don't have to register a clip in `sounds.xml` to play custom audio. Ship a
raw **PCM `.wav`** under the mod's `Resources/` and decode it at runtime into an
`AudioClip`, then play it through a Unity `AudioSource`. This sidesteps the whole
SoundDataNode/XML pipeline — handy for VFX-coupled SFX (thunder, weapon booms)
that don't need vanilla's sound categories/ducking.

- **Minimal WAV reader:** parse the RIFF header, accept `audioFormat == 1` (PCM)
  only, handle 8-bit unsigned / 16-bit signed, mono or stereo. Then
  `AudioClip.Create(name, frameCount, channels, sampleRate, false)` +
  `clip.SetData(samples, 0)`. (MP3 won't parse — ship `.wav`. The Unity build
  has no built-in MP3 decoder you can reach this way.)
- **3D placement:** an `AudioSource` on a throwaway GameObject; set
  `go.transform.position = worldPos - Origin.position` (logical→scene space),
  `spatialBlend` ~0.4 for a directional-but-not-pinpoint cue, then `Destroy(go,
  clip.length + delay + buffer)`.
- **Guard** with `if (GameManager.IsDedicatedServer) return;` — client-only.
- **Lazy-load** clips once from `Path.Combine(ModInstance.Path, "Resources",
  "<sub>")`; if the mod instance/path isn't ready yet, leave `_initialized=false`
  and retry on the next call.
- `deploy.ps1` copies the whole `Resources/` tree, so a new subfolder ships
  automatically (it's an *assets* change, but if the same deploy also rebuilds
  the DLL it becomes a restart deploy).

**Thunder realism trick (speed-of-sound):** detect each clip's loud-onset
("clap") time at decode (peak pass, then first 50 ms RMS window ≥30 % of peak).
At play time compute `perceivedDelay = clamp(dist/343, 0, 2)`; if
`perceivedDelay >= clapOnset` play the full pre-clap rumble after
`PlayDelayed(perceivedDelay - clapOnset)`, else scrub in with
`src.time = clapOnset - perceivedDelay` so a close strike claps instantly.
Reference: zPhone `src/apps/God/GodThunderSound.cs` (trimmed from the Scepters
mod's `PortalSounds`); used by the God app's Thunder + Kill-Everything buttons.
For mass effects (kill-all), play **one** clap for the whole volley, not one per
target, or you get a wall of overlapping claps.

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
