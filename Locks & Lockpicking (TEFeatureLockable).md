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

## Related

- Doors themselves: see "Composite Doors (TEFeatureDoor)" — `SetOpen`
  bypasses locks entirely when driving doors from code.
- World time: 24000 ticks/day, 1000/hour, so minutes = `minute * 1000 / 60`
  when calling `world.SetTime`.
