# Mod Structure

Part of the [7DTD Modding Knowledgebase](README.md). Covers folder layout, ModInfo.xml, DLL loading, and the modlet system.

---

## Folder Layout

```
<7DTD Install>/Mods/
└── YourModName/
    ├── ModInfo.xml           ← required — mod identity and metadata
    ├── YourMod.dll           ← compiled C# code (goes in mod root)
    ├── Config/
    │   ├── blocks.xml        ← XPath patch for block definitions
    │   ├── Localization.txt  ← translation strings
    │   ├── XUi/
    │   │   ├── windows.xml   ← XPath patch appending window definitions
    │   │   └── xui.xml       ← XPath patch registering window groups
    │   ├── XUi_Common/       ← shared control templates (available in all UI contexts)
    │   │   ├── controls.xml
    │   │   └── styles.xml
    │   └── XUi_Menu/         ← main menu UI modifications (separate from in-game UI)
    │       └── windows.xml
    ├── Resources/            ← Unity asset bundles (.unity3d) for custom audio/models/textures
    ├── Textures/             ← Raw PNG/texture files for UI (referenced as @modfolder:Textures/...)
    │   └── UILoadingTips/    ← Custom loading screen tip images
    ├── UIAtlases/
    │   ├── ItemIconAtlas/    ← custom item icon PNGs (game auto-discovers by filename)
    │   ├── UI/               ← custom UI sprites
    │   └── UIAtlas/          ← atlas textures
    ├── News/                 ← (optional) custom news/announcement XML
    ├── Music/                ← (optional) background music files
    └── Images/               ← (mod-specific) runtime assets (PixelPaste-specific)
```

**Notes:**
- `XUi/` patches the in-game UI; `XUi_Menu/` patches the main menu UI — they are separate contexts
- `XUi_Common/` is loaded in both contexts; use it for controls/styles shared across both
- `UIAtlases/ItemIconAtlas/` PNG files are named to match item names — the game auto-maps them as item icons
- `Resources/` `.unity3d` bundles are referenced from XML as `#@modfolder:Resources/file.unity3d?AssetName`

---

## ModInfo.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xml>
    <Name value="YourModName" />
    <DisplayName value="Your Mod Display Name" />
    <Description value="Short description." />
    <Author value="YourName" />
    <Version value="1.0.0" />
    <Website value="" />
    <SkipWithAntiCheat value="true" />
</xml>
```

> Note: `SkipWithAntiCheat value="true"` marks the mod as requiring EAC to be disabled. Set this for any mod that includes a C# DLL. The game will skip loading the mod (rather than crash) when EAC is active.

> **`<Name>` must match `^[0-9a-zA-Z_\-]+$`** — letters, digits, underscore, and hyphen only. **No spaces.** A space (or any other character) makes the loader log `ERR [MODS] <Folder>/ModInfo.xml does not define a valid non-empty Name … ignoring` and skip the ENTIRE mod (no items.xml merged, no DLL loaded, no localization). The mod folder name on disk can still have spaces — only the `<Name>` element value is regex-checked. Put any pretty/spaced label in `<DisplayName>` instead. Symptom: "I deployed the mod and nothing appears in-game" with no items.xml / sounds.xml load errors anywhere — they don't appear because the mod was rejected before XML merge ever ran.

---

## DLL Loading

- The compiled `.dll` goes in the **mod root** (same folder as `ModInfo.xml`)
- The game automatically discovers and loads it at startup
- **EAC (Easy Anti-Cheat) must be disabled** to load C# code mods — launch via the non-EAC shortcut or Steam launch option `-noeac`
- The DLL targets **.NET Framework 4.6.1**
- Reference the game's managed DLLs from `7DaysToDie_Data/Managed/`:
  - `Assembly-CSharp.dll` — main game logic
  - `UnityEngine.dll` — Unity base
  - `UnityEngine.CoreModule.dll`
  - `UnityEngine.ImageConversionModule.dll` — for `Texture2D.LoadImage`
  - `UnityEngine.InputLegacyModule.dll` — **required for the legacy `Input` class** (`Input.GetKey`, `Input.GetKeyDown`, etc.). Modern Unity splits `Input` out of CoreModule into this module; `KeyCode` resolves from CoreModule but `Input` does **not**. Without this reference you get `CS0103: The name 'Input' does not exist in the current context`.
  - `LogLibrary.dll` — **defines the `Log` class** (`Log.Out`/`Log.Warning`/`Log.Error`). `Log` is *not* in `Assembly-CSharp`; without this reference you get `CS0103: The name 'Log' does not exist in the current context`.
  - `UnityEngine.IMGUIModule.dll` — **required for a legacy `OnGUI` overlay** (`GUI`, `GUIStyle`, `GUI.DrawTexture`, `GUI.Label`, `GUI.Button`). Without it: `CS1069: 'GUIStyle' … forwarded to assembly 'UnityEngine.IMGUIModule'`. See [[IMGUI Tracker Stack]].
  - `UnityEngine.TextRenderingModule.dll` — **required alongside IMGUI for `GUIStyle` text config**: `FontStyle` and `TextAnchor` live here (`CS0012: 'FontStyle'/'TextAnchor' is defined in an assembly that is not referenced … UnityEngine.TextRenderingModule`).

> **Target `netstandard2.1` when the build machine has no .NET Framework targeting pack** (e.g. only the dotnet 8/10 SDK, no `Reference Assemblies/Microsoft/Framework/.NETFramework`). An SDK-style `.csproj` with `<TargetFramework>netstandard2.1</TargetFramework>`, the game DLLs as `<Reference>` with `<Private>false</Private>`, builds with plain `dotnet build` and the resulting DLL loads in the game's Mono runtime (only touches APIs the runtime provides). **Use `netstandard2.1`, not `2.0`:** the game DLLs reference `netstandard 2.1.0.0`, so a `netstandard2.0` project fails with **CS1705** ("Assembly 'Assembly-CSharp' … uses 'netstandard, Version=2.1.0.0' which has a higher version than referenced assembly 'netstandard, Version=2.0.0.0'"). The `MSB3277` version-conflict warnings against Unity module DLLs are noise — ignore them. Verified with Uncover.

> **Overriding game methods with `ReadOnlySpan<char>` params requires compiling against the game's own BCL.** Since the ~2026-06-29 game update, virtual signatures like `Entity.AllowActivationCommand(ReadOnlySpan<char>, EntityPlayerLocal)` use `ReadOnlySpan<T>`, and `Assembly-CSharp`'s typeref for it points at **Unity's `mscorlib 4.0.0.0`** (not `netstandard`). A `net48` target has no Span at all (CS0234), and a plain `netstandard2.1` target resolves `ReadOnlySpan` from the SDK ref pack's `netstandard.dll` — a *different type identity*, so the override still fails with **CS0115 "no suitable method found to override"**. Fix: keep `<TargetFramework>netstandard2.1</TargetFramework>` but add `<NoStdLib>true</NoStdLib>` + `<DisableImplicitFrameworkReferences>true</DisableImplicitFrameworkReferences>` and reference the game's `mscorlib.dll`, `netstandard.dll`, `System.dll`, `System.Core.dll` from `7DaysToDie_Data\Managed` as `<Reference>` items with `<Private>false</Private>`. Verified with Oppressor (2026-07-09). To check which assembly a typeref points at, read the metadata with `System.Reflection.Metadata` (`PEReader` → `TypeReferences` → `ResolutionScope`) — runtime reflection lies because .NET 8 unifies corlibs on load.

> **`ConsoleCmdAbstract` overrides must be `public`, not `protected`.** Adding a console command from a mod needs no Harmony — `SdtdConsole` finds every `IConsoleCommand` implementor across all loaded assemblies by reflection, so a public class with a parameterless ctor extending `ConsoleCmdAbstract` auto-registers. But the shipped `Assembly-CSharp` is *publicized*: `getCommands`/`getDescription`/`getHelp` carry `[PublicizedFrom(EAccessModifier.Protected)]` and their real metadata is `public`. Decompilers print `protected`, but declaring `protected override` fails with **CS0507 "cannot change access modifiers when overriding"** — use `public override`. Commands are typed in the F1 console without a leading slash. See [[Map Fog & Reveal]].
  - `UnityEngine.AnimationModule.dll` — for `Animator` (e.g. stamping door
    animator state). General rule: `UnityEngine.dll` is only a **facade** that
    type-forwards to the per-feature module DLLs; the compiler error
    `CS1069: The type name 'X' could not be found in the namespace
    'UnityEngine'. This type has been forwarded to assembly '...Module'`
    names the exact module DLL to add from `Managed/` (reference it with
    `<Private>false</Private>` like the others).

> **Numpad / NumLock gotcha:** On Windows, when NumLock is **off** the numeric keypad does not emit `KeyCode.Keypad*` at all — the OS sends the navigation keys instead (numpad 8→`UpArrow`, 2→`DownArrow`, 4→`LeftArrow`, 6→`RightArrow`, 9→`PageUp`, 7→`Home`, 1→`End`, 3→`PageDown`, 0→`Insert`, `.`→`Delete`; numpad 5 sends `VK_CLEAR`, which has no usable Unity `KeyCode`). So `Input.GetKeyDown(KeyCode.Keypad6)` silently never fires for a user with NumLock off — no error, key just does nothing. Symptom: "the numpad controls don't work." Fix: accept both the `Keypad*` code and its nav-key alias for each binding, and never use `Keypad5` as a modifier (pick `LeftShift`/`RightShift` instead). Airstrike's designator laser-origin tuner (`AirstrikeLaserOrigin.HandleAdjustInput`) does this.

> **`OnHoldingUpdate` is a throttled tick, not a per-frame callback:** `ItemAction.OnHoldingUpdate` does **not** fire every rendered frame, so polling `Input.GetKeyDown` inside it silently drops most presses — `GetKeyDown` is true for only one frame per press, and that frame usually lands between the throttled calls. Symptom: "pressing several times only moves once, then a long delay before it'll respond again." Fix: poll input from a real Unity `Update()`. Attach a small `MonoBehaviour` to a GameObject you already own (e.g. the laser object) and do the `Input` polling there; keep `OnHoldingUpdate` for visuals only. Airstrike's designator does this via `AirstrikeLaserTuner : MonoBehaviour`. (`Input.GetKey`/held-state polling tolerates the throttle, but edge detection via `GetKeyDown` does not.)

> **Silent-deploy gotcha:** `deploy.ps1` runs `build.ps1` first and aborts the whole deploy (robocopy never runs) if the build fails. A compile error therefore leaves the **old DLL** in the Mods folder, and modman's `status` stays `yellow` with `deployedId` unchanged. Symptom: "I deployed and restarted but my code change does nothing." Always confirm the build actually succeeded — check that the root `Airstrike.dll` timestamp is fresh, or grep the DLL bytes for a new symbol name.

> **Running-game DLL lock:** If 7DTD is running with a mod DLL loaded,
> `robocopy` can fail replacing `<GameDir>\Mods\<Mod>\<Mod>.dll` with
> `ERROR 1224 (0x000004C8) ... user-mapped section open`. Source build may
> still succeed and non-DLL files may copy, but the deployed DLL remains old.
> Fully quit the game, redeploy, then relaunch; `xui reload` cannot load a new
> DLL.

---

## Entry Point

Implement `IModApi` to get a callback when the mod loads:

```csharp
public class MyMod : IModApi
{
    public void InitMod(Mod _modInstance)
    {
        // _modInstance.Path = absolute path to the mod folder
        string modPath = _modInstance.Path;
        Log.Out("[MyMod] Loaded from: " + modPath);
    }
}
```

Only one class in the DLL needs to implement `IModApi`. The game finds and calls it automatically.

---

## Loading Order

The game loads mods in this sequence:

1. **Discover mods** — scans `Mods/` directory
2. **Load DLLs** — loads each mod's assembly
3. **Call `InitMod()`** — invokes `IModApi.InitMod()` for each mod
4. **Load Config XMLs** — parses and merges all `Config/*.xml` patches (blocks, items, etc.)
5. **Load Localization** — reads `Config/Localization.txt` from each mod
6. **Initialize world** — loads or creates the game world

**Key insight:** Since `InitMod()` runs *before* Config XMLs and Localization are loaded, mods can **dynamically generate** config files during initialization. This enables items, blocks, or localization entries that depend on runtime conditions (e.g., user-provided content files) rather than being hardcoded at build time.

---

## XML Config Patching

All XML files in `Config/` are XPath patch files — they modify vanilla game XML rather than replacing it. See [[XML Patching (XPath)]] for the full reference.

---

## Localization

Translation strings live in `Config/Localization.txt`. See [[Localization]].

---

## Custom UI Sprites (UIAtlases)

A mod can ship XUi sprites without an asset bundle: every PNG dropped in
`UIAtlases/UIAtlas/<name>.png` is loaded at startup into the in-game `UIAtlas`
as a sprite named `<name>`, usable directly as `sprite="<name>"` in XUi XML.
Plain quads — no slicing metadata, so `type="sliced"` only makes sense for
sprites drawn with uniform borders. New/changed PNGs are **xui-reload kind**
(no restart). The Elevator mod generates its panel art deterministically with
`tools/gen_ui_sprites.ps1` (run under Windows PowerShell 5.1, not pwsh —
System.Drawing).

---

## Build & Deploy

See `build.ps1` in the project root. After any code or XML change, always run:
```powershell
powershell -ExecutionPolicy Bypass -File build.ps1
```
The game loads from its own `Mods/` directory — the project directory is not used at runtime.

### ModForge Deploy Marker

For ModForge-managed workspaces, a new or repaired mod scaffold is not considered ready until it has a root-level `deploy.ps1`. Codex sessions should not run deploy scripts or launch the game directly. To request deployment, create or touch the mod root marker:

```powershell
New-Item -ItemType File -Force -Path "<ModRoot>\.modforge-deploy" | Out-Null
```

ModForge watches for that marker and runs deployment outside the coding session.
If `Test-Path` is false immediately after a successful marker touch, ModForge may have already consumed/deleted the marker to start queued deployment; do not use `Get-Item` as the marker existence check.

`.modforge-state.json` stores ModForge's per-kind drift counters as numeric strings under `kinds.restart`, `kinds.xui`, and `kinds.assets`. If a merge leaves conflict markers inside those values, resolve each conflicted counter to the highest numeric side so the counter remains monotonic and the file stays valid JSON.

### Build Script Path Resolution

Root-level `build.ps1` wrappers should not assume the process working directory is the mod root. Resolve the root from `MODFORGE_MOD_DIR` when present, otherwise from `$MyInvocation.MyCommand.Path`/`$PSScriptRoot`, and normalize any `\\?\` extended-length path prefix before using `Push-Location` or passing paths to tools.

Build wrappers should also map the common 7DTD install variables before invoking MSBuild:

```powershell
$gameDir = if ($env:GameDir) { $env:GameDir } elseif ($env:SEVEN_DTD_CLIENT) { $env:SEVEN_DTD_CLIENT } elseif ($env:SEVEN_DAYS_TO_DIE_PATH) { $env:SEVEN_DAYS_TO_DIE_PATH } else { "<default install>" }
dotnet build $project -c Release "/p:GameDir=$gameDir"
```

Project files can support the same convention by accepting `GameDir`, `SevenDaysToDiePath`, `SEVEN_DTD_CLIENT`, and `SEVEN_DAYS_TO_DIE_PATH`, then validating that `$(GameDir)\7DaysToDie_Data\Managed\Assembly-CSharp.dll` and required mod DLLs such as `Mods\0_TFP_Harmony\0Harmony.dll` exist before `ResolveReferences`.

> **Wrong-`GameDir` build produces a misleading flood of CS0246 errors.** If a `dotnet build` runs with a `GameDir` that doesn't exist on the machine, every `<Reference>` HintPath points at a missing DLL and is silently dropped, so the compiler reports `CS0246: 'Color'/'Vector3'/'GUIStyle'/'Input' could not be found` across **all** files — including ones that have always compiled. The errors look like a code problem but the real cause is unresolved game references. Fix: pass the actual install dir. On this workstation the game is on the D: Steam library, not the csproj's `C:\Program Files (x86)` default:
> ```bash
> dotnet build src/FPV.csproj -c Release "-p:GameDir=D:\SteamLibrary\steamapps\common\7 Days To Die"
> ```
> (Find it from `steamapps/libraryfolders.vdf`, or just check for `<lib>/steamapps/common/7 Days To Die/7DaysToDie_Data/Managed/Assembly-CSharp.dll`.) modman's own deploy build already knows the right path; this only bites local command-line builds.

> **Stale second install produces misleading CS0115 errors.** This workstation
> also has an **older copy at `D:\7 Days To Die`** (not the Steam library one).
> Building — or `ilspycmd`-decompiling — against it resolves references fine
> but the old `Block` virtual signatures don't match, so long-stable files
> fail with `CS0115: no suitable method found to override`
> (`OnBlockActivated`, `GetBlockActivationCommands`, …). The errors look like
> a code regression; the real cause is the wrong Assembly-CSharp. The live
> install is `D:\SteamLibrary\steamapps\common\7 Days To Die` (2026-07;
> confirm against `mcp__modman__status`'s `deployedPath`).

### Mod Gitignore Unity Entries

Always include these Unity workspace ignores in newly-created or repaired mod `.gitignore` files. The root-level hand-authored mod solution/project files and `src/` project are kept; only Unity-generated files under `UnityProject/` are ignored. If Unity has also created root-level `Library/`, `ProjectSettings/`, or `Temp/` folders in a mod whose real Unity project is under `UnityProject/`, treat those root folders as generated noise and ignore them too.

```gitignore
# Unity editor — all regenerated from Assets/ + ProjectSettings/
UnityProject/Library/
UnityProject/Logs/
UnityProject/Temp/
UnityProject/Obj/
UnityProject/obj/
UnityProject/UserSettings/
UnityProject/MemoryCaptures/

# Unity auto-generated IDE files (the hand-authored mod sln/csproj
# at the repo root + src/ are kept; this only catches UnityProject/)
UnityProject/*.csproj
UnityProject/*.sln
UnityProject/*.user

# Unity build artifacts
UnityProject/*.apk
UnityProject/*.aab
UnityProject/*.unitypackage
UnityProject/headless_build.log
```

---

## Dedicated Server Deployment

The dedicated server is a **separate Steam app** with its own install directory (typically `7 Days To Die Dedicated Server/`). It has its own `Mods/` folder that is independent of the client game's `Mods/` folder.

**Critical:** When connecting to a dedicated server, the server sends the block/item ID map to clients. **Clients receive block definitions from the server, not from their own local config files.** If a block is defined in the client's mod but not in the server's mod, the client will never see it — it won't appear in the creative menu, `giveself` won't work, and it can't be placed.

### Deploy to Both

Any mod that defines blocks, items, recipes, or localization must be deployed to **both** locations:

```
Client:  <7DTD Install>/Mods/YourMod/
Server:  <7DTD Dedicated Server Install>/Mods/YourMod/
```

All `Config/` XML files and `Localization.txt` must match between client and server. DLLs should also be deployed to both (server needs them for block classes, Harmony patches, etc.).

### Starting the Dedicated Server

The dedicated server must be started from its own install directory — `startdedicated.bat` resolves paths relative to its working directory:

```bash
cd "<7DTD Dedicated Server Install>"
./startdedicated.bat
```

Starting it from a different working directory will cause config loading failures.
