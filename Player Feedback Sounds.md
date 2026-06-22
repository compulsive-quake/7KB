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
- **Build gotcha — reference `UnityEngine.AudioModule`:** `AudioClip` and
  `AudioSource` are type-forwarded out of `UnityEngine.dll` into
  `UnityEngine.AudioModule.dll`. A mod `.csproj` that references only
  `UnityEngine` + `UnityEngine.CoreModule` fails to compile with **CS1069**
  ("type 'AudioClip' has been forwarded to assembly 'UnityEngine.AudioModule'").
  Add `<Reference Include="UnityEngine.AudioModule"><HintPath>$(ManagedDir)\
  UnityEngine.AudioModule.dll</HintPath><Private>false</Private></Reference>`.
  This failure mode is *silent in-game*: the `.wav` assets still deploy (assets
  bucket) and a stale prior DLL stays in place, so the feature looks shipped but
  the code that plays the audio never runs. Always confirm the DLL timestamp
  advanced and the new class name is present in the built DLL after adding audio.
  (Same trap applies to any UnityEngine type forwarded to a `*Module.dll` —
  reference the specific module assembly.)

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

## Looping "while-equipped" weapon hum (no native item property)

7DTD has **no item property for a sound that loops the whole time a weapon is
selected**. `Action0`'s `Sound_start`/`Sound_loop` only play while the trigger
is held (firing), not at idle. To reproduce a Quake-style idle hum/whir
(Gauntlet, Lightning Gun, Railgun, BFG `*_hum` clips), drive a looping Unity
`AudioSource` yourself off the locally-held item:

- A `DontDestroyOnLoad` MonoBehaviour polls the held item each `Update`:
  `GameManager.Instance.World.GetPrimaryPlayer().inventory.holdingItem` returns
  the current `ItemClass` (`holdingItem` is `public virtual ItemClass`, falls
  back to the bare-hand item when nothing's drawn); `.GetItemName()` gives the
  items.xml name to look up in a name→clip map.
- Only swap clips on change (compare to the currently-playing key) — don't
  restart the loop every frame.
- One `AudioSource` with `loop=true`, `spatialBlend=0` (2D — it's in the
  player's own hands). Guard `Create()` with `if
  (GameManager.IsDedicatedServer) return;`.
- Ship the clips as raw 16-bit PCM `.wav` and decode at runtime (see section
  above) so **no asset-bundle rebuild** is needed — adding a hum is just a
  `.wav` under `resources/sounds/` + a map entry. Convert with
  `ffmpeg -ar 44100 -ac 1 -c:a pcm_s16le`.
- Caveat: a raw `AudioSource` isn't routed to 7DTD's SFX mixer group, so it
  follows the global `AudioListener` volume but not the in-game "Sound" slider
  specifically. Keep the volume subtle (~0.45).

Reference: Quake Weapons mod `src/QuakeHum.cs` (`QuakeWeaponHum` + `WavLoader`),
wired from `ModApi.InitMod` alongside the Harmony patches.

## Quake 3 weapon loop mapping — readySound vs firingSound (Quake Weapons mod)

Quake 3 gives each weapon up to three looping/one-shot sounds in
`cg_weapons.c` (`CG_RegisterWeapon`), and they are **not** named the way you'd
guess from the filename:

- `readySound` — loops the whole time the weapon is **equipped but not firing**.
- `firingSound` — loops only **while the trigger is held** (replaces readySound
  via `if ( (eFlags & EF_FIRING) && firingSound ) … else if ( readySound ) …`).
- `flashSound[0]` — the per-shot muzzle one-shot.

The trap: the lightning gun's idle hum is **`sound/weapons/melee/fsthum.wav`**
— the *same clip as the gauntlet's* readySound — while
`sound/weapons/lightning/lg_hum.wav` is its **firingSound** (the electric
crackle), *not* the equipped hum. Quake Weapons' `QuakeHum.cs` plays while the
weapon is **selected**, so it mirrors `readySound`: map `quakeLightningGun` to
fsthum, not lg_hum. (Originally shipped lg_hum as the idle hum, which sounded
wrong — it's the beam loop.) `tools/convert_sounds.ps1` `$hums` now sources
fsthum for both gauntlet and lightning, so the two idle wavs are byte-identical.

`QuakeHum.cs` owns the LG's **entire** fire audio (strike + loop), not just the
idle hum. Three runtime clips, all decoded from raw PCM under resources/sounds/:
`ReadyHums` (fsthum, while selected & idle), `FiringLoops` (lg_hum, the sustained
beam crackle while held), `FireStarts` (lg_fire, a one-shot on the not-firing →
firing edge). On the firing edge it `PlayOneShot`s the strike on a second
AudioSource and swaps the sustained source fsthum→lg_hum; on release it swaps
back. Swap the sustained loop, don't layer — Quake's ready/firing are mutually
exclusive. lg_hum ships as `lightninggun_beam.wav`, lg_fire as
`lightninggun_strike.wav` (convert_sounds.ps1 `$firingLoops` / `$fireStarts`).

**Detect "trigger held" via `bPressed`, NOT a fire-time window.** First attempt
inferred firing from "a round fired within FiringHangtime (0.12 s)". That breaks
on a *discrete-round* weapon whose round interval exceeds the window: between
rounds the state falls back to "not firing", so the strike re-fires and the loop
restarts every shot — a stuttering, delayed mess (the LG is built on the SMG, so
it's discrete rounds under the hood, not a true held beam). The robust signal is
the live trigger flag: `player.inventory.holdingItemData.actionData[0]` cast to
`ItemActionRanged.ItemActionDataRanged`, then `.bPressed` — set true on fire-start
and false only on trigger release, so it stays true for the whole hold regardless
of per-round cadence. Note the data type is **nested** (`ItemActionRanged.`-
prefixed) and `actionData[0]` is the base `ItemActionData`, so an `as` cast is
required. This replaced the Harmony `fireShot` postfix entirely — no patch needed
for firing state, just read the held item's action data each `Update()`.

**Why C# owns it — the 7DTD firing-sound trap.** Decompiling
`ItemActionRanged.Fire` (called once per round): when `itemValue.Meta == 0`
(true for these guns) and the weapon is firing, it does
`Manager.Play(holdingEntity, SoundStart)` **every round** (plus a
`(_firingState==1) ? SoundStart : SoundLoop` per-round play). So `Sound_start`
itself replays per round — for a 1200 RPM gun the 2.77 s `lg_fire` stacks into a
continuous "strike wall" (SoundDataNode allowed `MaxVoices=30`,
`MaxRepeatRate=0.001`). There is **no XML way** to make it play once: removing
`Sound_loop` doesn't help (it just inherits the parent SMG's `smg_fire`, and the
per-round `Sound_start` replay remains). Fix used: **blank both** in the LG's
`Action0` (`value=""` — vanilla's own idiom for killing inherited sounds, e.g.
`Sound_reload`/`Sound_end value=""` on stock guns) so vanilla plays nothing, and
let QuakeHum.cs play lg_fire once + loop lg_hum. Side effect: a blanked weapon
sound emits **no AI noise** (the SoundDataNode's `<Noise>` never triggers) — the
LG is silent to zombies unless you re-add a noise source. Leave the machineguns'
`Sound_loop` alone — per-round retrigger is correct for discrete-shot full-auto;
only continuous-beam weapons need this C#-owned once-then-loop shape.

Reference: `code/cgame/cg_weapons.c` in id-Software/Quake-III-Arena (GitHub);
`ItemActionRanged.Fire` in 7DTD `Assembly-CSharp.dll`.

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
