# 7DTD 3.0.0 API Changes (Mod Migration)

Breaking C# API changes observed when migrating mods from 7DTD 2.x to **3.0.0**.
A mod DLL compiled against 2.x loads under 3.0.0 but throws `MethodAccessException` /
`Method not found` at runtime for any removed overload. **Recompile against the 3.0.0
`Assembly-CSharp.dll` to surface these as compile errors**, then fix per below.

## Symptom: silent feature break, `Method not found` in log

A 2.x DLL that calls a now-removed overload fails only when that code path runs. If the
call is inside a try/catch (e.g. an input listener's `Update`), you'll just see a warning
like:

```
WRN [zPhone] Listener update error: Method not found: void .GUIWindowManager.Open(string,bool,bool,bool)
```

The 4-arg `Open` doesn't exist in source — the 2.x compiler **baked optional-parameter
defaults into the call site**, so a 2-arg source call (`Open("x", true)`) compiled to a
4-arg IL call. When 3.0.0 changes the optional params, the baked call no longer resolves.
Recompiling re-binds it to the current overload.

## `World` — `clrIdx` (cluster index) first parameter removed

3.0.0 dropped the leading `int _clrIdx` argument from many `World` block/tile methods
(continues the same cleanup as the `ExplosionServer` `clrIdx` removal). Block position now
goes through `BlockValueRef`, which has an **implicit conversion from `Vector3i`** (and a
`new BlockValueRef(Vector3i)` ctor).

| 2.x call | 3.0.0 call |
|---|---|
| `world.GetTileEntity(0, pos)` | `world.GetTileEntity(pos)` — `GetTileEntity(Vector3i)` or `(BlockValueRef)` |
| `world.SetBlockRPC(0, pos, bv)` | `world.SetBlockRPC(new BlockValueRef(pos), bv)` — `SetBlockRPC(BlockValueRef, BlockValue)` |

## `World.ChunkClusters` collection removed → `World.ChunkCache`

The indexed `world.ChunkClusters[i]` list is gone. There's now a single primary
`public ChunkCluster ChunkCache` field. Replace the iterate-all-clusters pattern:

```csharp
// 2.x
for (int i = 0; i < world.ChunkClusters.Count; i++) {
    var ebcd = world.ChunkClusters[i].GetBlockEntity(pos);
    ...
}
// 3.0.0
var ebcd = world.ChunkCache?.GetBlockEntity(pos);   // ChunkCluster.GetBlockEntity(Vector3i)
```

## `XUiController.RefreshBindings(bool)` → `RefreshBindings()`

The `bool` overload (force/release) was removed; only the no-arg `RefreshBindings()` remains.
Drop the argument at every call site.

## `GUIWindowManager.Open` overloads

Current overloads (no 4-arg form):

```
Open(string _windowName, bool _bModal)
Open(string _windowName, bool _bModal, bool _bIsNotEscClosable)
Open(GUIWindow _w, bool _bModal)
Open(GUIWindow _w, bool _bModal, bool _bIsNotEscClosable)
```

Source like `windowManager.Open("zPhoneHome", true)` is still correct — it binds to the
2-arg overload once recompiled.

## XUi: vanilla folder split — mods must move `Config/XUi/` → `Config/XUi_InGame/`

**This is the big one and easy to miss.** 2.x loaded a single `Data/Config/XUi/` folder.
3.0.0 split vanilla XUi into three folders: **`XUi_Common`**, **`XUi_InGame`**, **`XUi_Menu`**
(confirmed as metadata strings in `Assembly-CSharp.dll`; the old `Config/XUi/` is gone).

7DTD's modlet patch system applies a mod file by **mirroring the vanilla file's exact relative
path**. So a mod that shipped `Config/XUi/windows.xml` + `Config/XUi/xui.xml` patched the 2.x
vanilla `Config/XUi/*`. In 3.0.0 there is no `Config/XUi/` to mirror, so those mod files patch
**nothing** — and there's no error. `xui reload` even logs `Parsing all window groups completed`
because it parsed the (now-orphan) document; the window group still never reaches the in-game
`GUIWindowManager`. Net symptom: `Window "<name>" unknown!` at `Open()` time even though the
files look loaded.

Fix — relocate the mod's in-game UI files to mirror the new path:

```
Config/XUi/windows.xml  ->  Config/XUi_InGame/windows.xml
Config/XUi/xui.xml      ->  Config/XUi_InGame/xui.xml
```

(In-game HUD/gameplay windows → `XUi_InGame`; main-menu UI → `XUi_Menu`; shared styles/controls
→ `XUi_Common`.) Vanilla roots are unchanged: `XUi_InGame/windows.xml` root is `<windows>`,
`XUi_InGame/xui.xml` root is `<xui>` — so `xpath="/windows"` and `xpath="/xui"` appends still work.

### `<ruleset>` wrapper also removed from `xui.xml`

2.x wrapped window-group registration in `<xui><ruleset name="default">...`. **3.0.0 removed
the `<ruleset>` element** — `<window_group>` nodes are now direct children of the root `<xui>`.
Any modlet that registered a custom window group with:

```xml
<append xpath="/xui/ruleset[@name='default']">   <!-- 2.x: no longer matches anything -->
```

silently no-ops in 3.0.0 (the xpath matches nothing, no error), so the window group never
registers. Symptom at runtime — opening the window logs:

```
WRN GUIWindowManager.Open: Window "<name>" unknown!
```

Fix: append directly to `/xui`:

```xml
<append xpath="/xui">
    <window_group name="..." controller="...">
        <window name="..." />
    </window_group>
</append>
```

`windows.xml` is unchanged — window *definitions* still append to `/windows`. Note the split:
vanilla XUi now lives under `Data/Config/XUi_InGame/`, `XUi_Menu/`, `XUi_Common/` (the old
single `Data/Config/XUi/` folder is gone), but mods still ship their own `Config/XUi/` folder.

This is an XUi-only change → hot-reloadable (`xui reload`), no DLL rebuild or game restart.

## Localization: `Config/Localization.txt` → `Config/Localization.csv`

3.0.0 renamed the per-mod localization file. `Localization.LoadPatchDictionaries`
hardcodes `Localization.csv`; a mod still shipping `Localization.txt` gets **no error,
no log line** — item/entity tooltips just show raw keys (`vehicleFooPlaceable`).
Rename (or ship both for 2.x compat) and **restart** the game — mod localization is
only loaded at startup (`ModManager.LoadLocalizations` bails when `_isLoadingInGame`).
Column mapping is by header name, so old headers without 3.0.0's `KeepLoaded` column
still work. Details in [Localization](Localization.md).
## Block activation methods also lost `_cIdx` (and the `_interaction` string)

The `Block` activation surface dropped the cluster-index parameter in 3.0.0 (same
cleanup as the `World` methods above). Current signatures (reflection-verified,
Elevator mod 2026-07):

```csharp
bool  OnBlockActivated(WorldBase _world, Vector3i _blockPos, BlockValue _blockValue, EntityPlayerLocal _player)
bool  OnBlockActivated(string _commandName, WorldBase _world, Vector3i _blockPos, BlockValue _blockValue, EntityPlayerLocal _player)
bool  HasBlockActivationCommands(WorldBase _world, BlockValue _blockValue, Vector3i _blockPos, EntityAlive _entityFocusing)
BlockActivationCommand[] GetBlockActivationCommands(WorldBase _world, BlockValue _blockValue, Vector3i _blockPos, EntityAlive _entityFocusing)
string GetActivationText(WorldBase _world, BlockValue _blockValue, Vector3i _blockPos, EntityAlive _entityFocusing)
```

Note the old 2.x `OnBlockActivated(string _interaction, WorldBase, int _cIdx, ...)`
form (still visible in older mods like SentryLight) no longer overrides anything —
copying signatures from an un-migrated sibling mod produces methods that silently
never get called. Reflect on the current assembly instead.

## Block lifecycle signatures (3.0, reflection-verified)

`OnBlockAdded` gained a trailing `PlatformUserIdentifierAbs`; the other two are
unchanged (verified against the 3.0 assembly, Elevator mod 2026-07):

```csharp
void OnBlockAdded(WorldBase _world, Chunk _chunk, Vector3i _blockPos, BlockValue _blockValue, PlatformUserIdentifierAbs _addedByPlayer)
void OnBlockRemoved(WorldBase _world, Chunk _chunk, Vector3i _blockPos, BlockValue _blockValue)
void OnBlockUnloaded(WorldBase _world, Vector3i _blockPos, BlockValue _blockValue)
```

Gotchas when overriding `OnBlockRemoved` to clean up per-block mod data:

- It fires for **every** air-set, including your own `SetBlocksRPC` batches. A mod
  that moves blocks (like the Elevator car) must guard with "is this position part
  of my own move?" or it deletes its data mid-move.
- Check `_blockValue.ischild` and only act on the master of a multiblock.
- Chunk unload is `OnBlockUnloaded`, not `OnBlockRemoved` — unload must NOT drop
  persistent data.

## `GameManager.ShowTooltip` is static now

`GameManager.Instance.ShowTooltip(...)` → CS0176. Call it as
`GameManager.ShowTooltip(EntityPlayerLocal _player, string _text, bool _showImmediately, bool _pinTooltip, float _timeout)`.

## `Chunk.RemoveTileEntityAt` is generic

`chunk.RemoveTileEntityAt(world, localPos)` no longer infers —
use `chunk.RemoveTileEntityAt<TileEntity>(world, localPos)`.

## `ModEvents` handlers take typed ref-struct data

Every `ModEvents` event is a `ModEvent<TData>` whose handler signature is
`void Handler(ref ModEvents.S<Name>Data _data)`:

```csharp
ModEvents.GameStartDone.RegisterHandler(OnGameStartDone);
static void OnGameStartDone(ref ModEvents.SGameStartDoneData _data) { ... }
```

Also new and very useful: **`ModEvents.UnityUpdate`** fires every Unity frame —
per-frame input polling / timers from a plain static class, no MonoBehaviour or
Harmony patch needed. Guard on `GameManager.Instance?.World != null` since it also
fires in the main menu.

## Migration checklist

1. Point the csproj `Assembly-CSharp` reference at the 3.0.0 `7DaysToDie_Data\Managed\`.
2. `dotnet build` and fix every error (the ones above are the common ones).
3. DLL change → full deploy + game **restart** (not xui reload) to take effect.
4. Quick way to discover a new signature without a decompiler — reflect on the shipped DLL:
   `[Reflection.Assembly]::LoadFrom("...\Assembly-CSharp.dll").GetType("World").GetMethods() | ? Name -eq 'GetTileEntity'`
