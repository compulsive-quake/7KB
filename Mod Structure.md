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

## Build & Deploy

See `build.ps1` in the project root. After any code or XML change, always run:
```powershell
powershell -ExecutionPolicy Bypass -File build.ps1
```
The game loads from its own `Mods/` directory — the project directory is not used at runtime.

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
