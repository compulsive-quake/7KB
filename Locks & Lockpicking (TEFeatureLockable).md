# Locks & Lockpicking (TEFeatureLockable / TEFeatureLockPickable)

How locking works in 3.0.x and where to patch to bypass it. Verified against
the 3.0.x Assembly-CSharp (2026-07, decomp at
`Elevator/.claude/tmp/decomp/`); working example: zPhone's
`src/apps/God/GodMasterKey.cs` ("Master Key" God-app toggle).

## Anatomy

The legacy `TileEntitySecure*` classes are **gone** in 3.0.x. All lockable
world blocks (doors, gates, garage doors, hatches, safes, storage chests,
signs) are composite tile entities carrying:

- `TEFeatureLockable : TEFeatureAbs, ILockable` — player locks + keypad
  passwords. State: `locked` bool, `allowedUserIds`, `passwordHash`
  (persisted in Read/Write).
- `TEFeatureLockPickable : TEFeatureAbs` — POI pick-locks. State:
  `unlockCompletion` (0..1) — **not persisted**, resets on chunk reload.
  Successful pick calls `DowngradeToUnlockedVariant` (swaps to
  `LockPickDowngradeBlock`, permanent).

Other `ILockable` implementers (separate code paths, not covered by the
feature patches): `TileEntityVendingMachine`, `EntityVehicle`, `EntityDrone`.

## Where activation is actually gated

- `TEFeatureDoor.OnBlockActivated` ("open"/"close"): denied when
  `lockFeature.IsLocked() && !lockFeature.IsUserAllowed(PlatformManager.InternalLocalUserIdentifier)`.
  Note: the door's `AllowBlockActivationCommand` does NOT check locks — only
  open/close state; the deny is inside OnBlockActivated.
- `TEFeatureStorage.OnBlockActivated` ("Search"): same
  `IsLocked() && !IsUserAllowed()` conjunction, plus `isJammed` (quest
  containers — separate mechanic, not a lock).
- `TEFeatureLockPickable.AllowBlockActivationCommand`: while
  `unlockCompletion != 1f`, disables **every other feature's commands**
  (storage "Search", door "open") for non-owners — leaving "pick" (order
  `First`) as the only tap-E action. This, not NeedsLockpicking(), is the
  real player-facing lockpick gate.
- `TEFeatureDoor.CanOpen(out canPickToOpen)` checks `IsLocked()` /
  `NeedsLockpicking()` but is only called by AI pathing
  (`TraversalProviderNoBreak`, `EntityMoveHelper`), not player activation.
- Command enabling is aggregated in `TileEntityComposite`: every feature's
  `AllowBlockActivationCommand(module, commandName, ...)` is asked about
  every command; first `false` wins. `_module` is the feature that OWNS the
  command being considered, so `_module == this` identifies a feature's own
  commands. Base impl (`TEFeatureAbs`) returns `true` unconditionally —
  skipping the original in a prefix on a subclass loses nothing.

## Bypassing locks globally (zPhone Master Key pattern)

Four Harmony patches, all read-through (no lock state mutated, instantly
reversible):

1. `TEFeatureLockable.IsLocked` prefix → `__result = false`. Covers door /
   storage denies, keypad flow, tooltips.
2. `TEFeatureLockable.IsUserAllowed` prefix → `__result = true`. Backstop:
   `IsLocked()` is a one-line getter the Mono JIT may inline into callers
   (bypassing the detour), but every player-facing deny is
   `IsLocked() && !IsUserAllowed()` and IsUserAllowed is too big to inline.
3. `TEFeatureLockPickable.NeedsLockpicking` prefix → `__result = false`
   (tooltips + AI CanOpen).
4. `TEFeatureLockPickable.AllowBlockActivationCommand` prefix →
   `__result = !ReferenceEquals(_module, __instance); return false;` —
   hides "pick" and re-enables Search/open, so tap-E opens POI safes
   directly.

Gotchas:

- `AllowBlockActivationCommand` has a `ReadOnlySpan<char> _commandName`
  param. A Harmony prefix works fine as long as the patch method does NOT
  declare the span param (Harmony only injects requested args; ref structs
  can't be boxed into `__args`).
- Don't set `unlockCompletion = 1f` instead of patching — it looks unlocked
  even after the toggle is switched off, until the chunk reloads.
- Patching `IsLocked` false also affects `CopyFromInternal` /
  `UpgradeDowngradeFrom`: a locked block upgraded while the bypass is on
  loses its locked state.
- `isJammed` (TEFeatureStorage, `PropIsJammed`) is a broken-lock quest state,
  not a lock — the patches above intentionally don't touch it.

## Bypassing a lock for your OWN block subclass (no Harmony needed)

Because the deny lives in `TEFeatureDoor.OnBlockActivated` and not in command
enabling, a `BlockCompositeTileEntity` subclass can intercept the activation
before `base` ever reaches the feature — no patch, no global effect:

```csharp
public override bool OnBlockActivated(string _commandName, WorldBase _world,
    Vector3i _blockPos, BlockValue _blockValue, EntityPlayerLocal _player)
{
    if (_commandName.EndsWith(":open", StringComparison.Ordinal) && MyConditionHolds())
    {
        // SetOpen on every TEFeatureDoor — ignores locks by design
        return true;
    }
    return base.OnBlockActivated(_commandName, _world, _blockPos, _blockValue, _player);
}
```

Key detail: command names reaching the Block for a composite are
**namespaced** — `TileEntityComposite.InitBlockActivationCommands` builds them
as `featureData.Name + ":" + command`, so you get `"TEFeatureDoor:open"`, not
`"open"` (`SplitFullCommandName` splits on the first `:`). `featureData.Name`
is the CompositeFeatures **class name** from blocks.xml, and two entries of the
same class (the vanilla elevator doors declare `TEFeatureDoor` twice) produce
the same prefix — so match on the suffix, not the whole string.

Also override `Block.GetActivationText` for the same condition: the "Locked"
line comes from the feature (`TileEntityComposite.GetActivationText` returns
the first non-null feature text, doors before lockable), so without it the
tooltip still says Locked while E works. Reuse the vanilla strings —
`string.Format(Localization.Get("tooltipUnlocked"), markup, Localization.Get("door"))`.
For `markup`, pass the literal `"[action:local:Activate][action:permanent:Activate]"`
rather than `playerInput.Activate.GetBindingXuiMarkupString()`: the tooltip
renderer expands it, and the binding call drags in a reference to the
`InControl` assembly (CS0012 if your csproj doesn't reference it).

Working example: `BlockElevatorDoor` + `ElevatorDoors.LockBypassed` /
`ToggleByHand` in the Elevator mod — a locked landing door opens by hand only
while the car is parked at that floor.

## Owner + ally checks for your OWN data (no tile entity involved)

When a mod keeps its own per-thing state (not in a TE) and wants vanilla-style
"owner / allies / only me" access control, the pieces are:

- **Local player's identity**: `PlatformManager.InternalLocalUserIdentifier`
  (`using Platform;`) → `PlatformUserIdentifierAbs`. Compare with
  `.Equals(other)`; the type implements value equality and `GetHashCode`, so it
  also works as a dictionary key.
- **Who placed a block**: `Block.OnBlockAdded(..., PlatformUserIdentifierAbs
  _addedByPlayer)`. `Chunk.SetBlock` fills it from
  `persistentPlayers.GetPlayerDataFromEntityID(_changedByEntityId).PrimaryId`
  and passes **null** when the change isn't attributed to an entity (world
  gen, some RPC paths) — always handle null rather than assuming a placer.
- **Ally test**: `GameManager.Instance.persistentPlayers.Allies.IsAlly(a, b)`
  (`AllyStore`), or the convenience `PersistentPlayerData.IsAlly(other)`.
  `AllyStore.GetStatus` returns `NotAllied` when either side is null, so no
  extra null guard is needed.
- **Display name**: `persistentPlayers.GetPlayerData(id)?.PlayerName
  .SafeDisplayName` — returns null for a player this world has never seen, so
  fall back to a generic word rather than the raw platform id.
- **Persisting an owner in a plain-text save**: `id.CombinedString` /
  `PlatformUserIdentifierAbs.FromCombinedString(s, _logErrors: false)`. The
  format is `Platform_UserId` — one token, no spaces, safe in a space-delimited
  line. Pass `_logErrors: false` or an id from a platform this client doesn't
  know spams the log on every world load.

Design notes that turned out to matter (Elevator mod's `ElevatorPermissions`):

- **Fail open on a null owner.** Data saved before the feature existed has no
  owner, and a restriction with nothing to compare against would lock everyone
  out. Treat "no owner" as unrestricted, and claim ownership the moment the
  player actually picks a restriction.
- **`dm` (`EnumGamePrefs.DebugMenuEnabled`) as the escape hatch** — vanilla
  `TEFeatureLockable.IsUserAllowed` does exactly this for non-door features, so
  a thing whose owner never comes back is still recoverable.
- Gate the **entry points** (`OnBlockActivated` cases, the Harmony postfix that
  opens a window) rather than the worker functions, so each refusal carries its
  own tooltip — and mirror the same test in `GetActivationText`, or the look-at
  prompt keeps promising an action that will be refused.
- These checks are client-local. Fine for an SP/local-host mod; not server
  authority — a dedicated-server build needs the check on the host too.

## Related

- Doors themselves: see "Composite Doors (TEFeatureDoor)" — `SetOpen`
  bypasses locks entirely when driving doors from code.
- World time: 24000 ticks/day, 1000/hour, so minutes = `minute * 1000 / 60`
  when calling `world.SetTime`.
