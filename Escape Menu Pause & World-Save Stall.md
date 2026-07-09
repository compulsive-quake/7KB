# Escape Menu Pause & World-Save Stall

Why "the game freezes for a second or two" the moment a mod hands the screen
back and the player's ESC press reaches the vanilla pause menu — and the
Harmony-free way to block it. Verified against the installed 3.0.0
`Assembly-CSharp.dll` (2026-07).

## The stall chain

ESC → NGuiAction "Menu" (built in `PlayerMoveController`) →
`windowManager.SwitchVisible(XUiC_InGameMenuWindow.ID)` →
`XUiC_InGameMenuWindow.OnOpen()` → `GameManager.Instance.Pause(true)`.

`Pause()` only records `requestedPauseState`; the work happens next
`GameManager.updatePauseState()`. In **single player** (server + pause
allowed) that does:

```
SaveLocalPlayerData();
SaveWorld();            // World.Save → ChunkCluster.Save → ChunkProvider.SaveAll
Time.timeScale = 0;
```

`SaveAll` ends in `RegionFileManager.WaitSaveDone()`, which **spins the main
thread** (`Thread.Sleep(20)` loop) until every queued dirty chunk is flushed
to region files. With a lot of dirty chunks — e.g. a mod `ChunkObserver` that
roamed across the map (FPV drone, vehicle autopilot, camera fly-through) —
this is a 1–2+ second hard freeze.

**Log tell:** `Saving 177 of chunks took 1762ms` followed by
`UnloadUnusedAssets after X m, took Y ms` right after your exit action means
the pause menu opened (even if you closed it a frame later — the save already
ran on open).

## Why "close it after it opens" doesn't work

Closing the menu next frame (`windowManager.Close("ingameMenu")`) restores
the UI but the save fired inside `OnOpen`. Suppression must prevent the
**open**.

## The Harmony-free gate: HUD FullHide

`GUIWindowManager.openInternal` early-outs when `IsFullHUDDisabled()`
(`bHUDEnabled == HudEnabledStates.FullHide`), and the Menu NGuiAction's
enabled-delegate is `!windowManager.IsFullHUDDisabled()`. So while your mod
holds `SetHUDEnabled(FullHide)` (typical for a full-screen IMGUI feed), the
pause menu **cannot** open — no Harmony needed.

The trap is restoring the HUD **in the same frame** you handle the ESC press:
the game's own Menu action processes that same press later in the frame (or
on key release), finds the HUD re-enabled, and opens the menu → pause → save.
Script-execution order decides who sees ESC first, so inline restore is a
coin flip.

**Fix:** defer the HUD restore in a coroutine until the key is fully released
plus one settle frame:

```csharp
private IEnumerator RestoreHudWhenEscapeClears(GUIWindowManager.HudEnabledStates prev)
{
    yield return null;
    while (Input.GetKey(KeyCode.Escape)) yield return null;
    yield return null;   // settle frame past the release edge
    LocalPlayerUI.GetUIForPrimaryPlayer()?.windowManager?.SetHUDEnabled(prev);
}
```

Exits not driven by ESC just wait the two settle frames (~30 ms, invisible).
Guard re-entry: if the mod re-engages (FullHide again) while a restore is
still pending, stop the pending coroutine and **keep** the previously captured
pre-mod HUD state — `wm.bHUDEnabled` still reads your own FullHide at that
point and must not be captured as "previous".

## The Harmony gate: skip the save only inside updatePauseState (zPhone "Fast Pause")

When the goal is to kill the stall for the *player's own* ESC (not a mod
handing the screen back), the HUD-FullHide trick doesn't apply — the menu
*should* open, just without the save. zPhone's God app ships this as the
"Fast Pause" toggle (`src/apps/God/GodFastPause.cs`):

- Prefix + Finalizer on `GameManager.updatePauseState` set/clear a static
  `InPauseUpdate` flag (main-thread only, no reentrancy — a plain bool is
  enough).
- Skip-prefixes (`return false`) on `GameManager.SaveWorld` and
  `GameManager.SaveLocalPlayerData` fire only when
  `toggle && InPauseUpdate`.

Scoping via the flag matters: `SaveWorld`/`SaveLocalPlayerData` are also
called from `SaveAndCleanupWorld` (quit path) and the periodic player-data
countdown — those must keep saving or you get real data loss on exit. With
the pause-save skipped, the world still saves on quit and on chunk unload;
the only cost is that a *crash* while paused loses progress since the last
save, same as a crash during play. Skip both saves together (not just
`SaveWorld`) so a crash can't restore player inventory inconsistent with an
older world state (dupe/void items).

Verified compiling against 3.0.0: `updatePauseState` is
`[PublicizedFrom(EAccessModifier.Private)]` → public in the shipped
assembly, patchable by name.

## Related perceived-freeze on view handoff

The other hitch on screen handoff is `SetFirstPersonView(true)` rebuilding
the SDCS player rig (see [Camera View Switching (First-Third Person)](Camera%20View%20Switching%20(First-Third%20Person).md)
and Items.md "held item churn"). It can't be avoided if you switched the
player third-person for the duration — but you can hide it: render one frame
of your full-screen transition overlay first, then run the heavy restore the
next frame (coroutine `yield return null`), so the stall lands behind a
static covering image instead of freezing live gameplay.
