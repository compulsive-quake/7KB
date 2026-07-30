# Composite Doors (TEFeatureDoor)

How to drive modern vanilla doors (open/close/state) from mod code, and how to
subclass them. Verified against the 3.0.x Assembly-CSharp (2026-07); working
example: the Elevator mod's `BlockElevatorDoor.cs` + `ElevatorDoors.cs`.

## Anatomy

Modern vanilla doors (`elevatorDoor`, `elevatorDoorDouble`,
`elevatorDoorTenthBlock`, commercial doors, …) are **not** `BlockDoorSecure`.
They use:

- `Class=CompositeTileEntity` → C# class `BlockCompositeTileEntity : Block`
- a `CompositeFeatures` property class listing features: `TEFeatureDoor`
  (open/close + animation + sounds), `TEFeatureLockable`,
  `TEFeatureLockPickable`, … — lock internals and how to bypass them are in
  "Locks & Lockpicking (TEFeatureLockable).md"
- door state lives on the `TileEntityComposite` tile entity, not in BlockValue
  meta.

Gotcha: the vanilla elevator doors declare **two** `TEFeatureDoor` entries on
the same block (one bare, one carrying the OpenSound/CloseSound props). When
driving a door from code, set state on **all** door features, not just
`GetFeature<TEFeatureDoor>()` (which returns the first).

## Driving a door from code

```csharp
TileEntityComposite te = world.GetTileEntity(masterPos) as TileEntityComposite;
if (te != null)
{
    ITileEntityFeature[] features = te.modulesInternalOrder; // public (publicized assembly)
    for (int i = 0; i < features.Length; i++)
    {
        TEFeatureDoor door = features[i] as TEFeatureDoor;
        if (door != null && door.IsOpen() != wantOpen)
        {
            door.SetOpen(wantOpen, true); // _animate: true → plays animation AND the door's own Open/CloseSound
        }
    }
}
```

- `SetOpen(open, _animate: true)` handles animation + sound itself
  (`HandleDoorAnimation` / `HandleOpenCloseSound`); with `_animate: false` the
  state flips silently and the anim snaps when the model syncs.
- `SetOpen` **bypasses lock checks** — `CanOpen` is only consulted for player
  activation. Fine for machinery driving its own doors.
- Use the **master** (parent) block position — the TE lives there. Doors are
  multiblocks (`MultiBlockDim 1,2,1` / `2,2,1`).
- Chunk not loaded → `GetTileEntity` returns null; just skip.
- The shipped Assembly-CSharp is publicized (`[PublicizedFrom]`), so
  `modulesInternalOrder`, `isOpen`, `autoCloseTime` etc. are directly
  accessible — no reflection needed.

## Door models instantiated outside the chunk pipeline play their entry animation

The door prefab's Animator entry state is the **Close clip from frame 0 —
whose first frame is the fully open pose**. When a chunk displays a door
block, `TEFeatureDoor.SetBlockEntityData` → `ForceAnimationState` stamps the
real state, so parked doors look right. But a door model you instantiate
yourself (e.g. via `ItemClassBlock.CreateMesh` for a moving proxy) is wired to
no tile entity: it visibly pops open and re-closes right after spawning.

Fix: replicate `ForceAnimationState` on the fresh transform (all hashes are
on the public `AnimatorDoorState : StateMachineBehaviour`):

```csharp
Animator[] anims = model.GetComponentsInChildren<Animator>(true);
for (int i = 0; i < anims.Length; i++)
{
    anims[i].enabled = true;
    anims[i].keepAnimatorStateOnDisable = true;
    anims[i].SetBool(AnimatorDoorState.IsOpenHash, open);
    anims[i].Play(open ? AnimatorDoorState.OpenHash : AnimatorDoorState.CloseHash, 0, 1f);
}
```

`Play(state, 0, 1f)` jumps to the clip's end frame — a silent snap, no sound
(sounds come from `TEFeatureDoor`, not animation events). Working example:
`ElevatorDoors.StampDoorState` in the Elevator mod, called from
`ElevatorMover.BuildProxy` for every captured model-entity block.

## `SetOpen(open, animate:true)` needs the model already displayed, or the animation is silently lost

`SetOpen(_animate: true)` → `HandleDoorAnimation(true)` → `UpdateAnimState`,
which grabs `world.ChunkCache.GetBlockEntity(masterPos)` and, **only if it has a
live transform** (`bHasTransform`), sets the `IsOpen` bool + fires the
`OpenTrigger` that plays the slide. If the block-entity model isn't displayed
yet, `UpdateAnimState` returns early — the animation request is **dropped
without error**. Then, frames later, the chunk instantiates the model
(`Chunk.OnDisplayBlockEntities` → `OnBlockEntityTransformAfterActivated` →
`TileEntityComposite.SetBlockEntityData` → `TEFeatureDoor.ForceAnimationState`)
which `Play(..., 1f)`-snaps the door to its final pose. Net effect: **the door
pops open with no slide** — even though you asked for `animate:true`.

This bites whenever you `SetOpen(true, true)` in the same frame you disturbed
the door's chunk. In the Elevator mod the car's arrival re-places the whole car
volume via `SetBlocksRPC` (a chunk remesh), then opened the arrival doors
immediately — so every re-placed (and same-chunk) door opened instantly.
Closing on **departure** looked fine because that runs while the door is still
displayed and its chunk is untouched.

Fix: **defer the open until the model is live again.** Queue the door master
positions, then each frame check `GetBlockEntity(pos)?.bHasTransform` and only
`SetOpen(true, true)` once it's true (with a few-second timeout fallback so a
door whose chunk stays unloaded still gets its state set). See
`ElevatorDoors.OpenAtArrival` + `DrivePendingOpens` (driven from
`ElevatorDoors.Update`). Note this is the inverse problem to the proxy-clone
snap above: there you *force* the snap on a model with no TE; here you must
*wait* for the model so the TE-driven animation isn't thrown away.

Don't over-apply the queue: if you haven't touched the door's chunk this frame
the model is already displayed, so `SetOpen(true, true)` animates straight away.
Only the deferred path needs the wait (`ElevatorDoors.OpenAtFloor` opens a
parked car's doors directly; `OpenAtArrival` queues, because a landing just
remeshed the chunk).

## Vanilla auto-close (and why you may want your own)

`TEFeatureDoor` has built-in auto-close: block property `AutoCloseTime`
(seconds, parsed in feature init) arms `autoCloseAtTickTime` inside `SetOpen`;
the server-side `UpdateTick` closes the door when the tick passes. Limits:

- It is a **block-type** property, not per-placed-door — no per-instance
  configuration.
- Setting `te.autoCloseTime` at runtime doesn't persist (re-read from block
  props on TE init) and only arms on the next open.

For per-door configurable auto-close, run your own timer (see
`ElevatorDoors.Update` in the Elevator mod: scan every 0.5 s, track
first-seen-open time per door pos, `SetOpen(false, true)` when elapsed).

## Subclassing a composite door

Extend `BlockCompositeTileEntity`, NOT `Block`, or you lose tap-E open/close,
locking, and the door radial. To add a custom radial command:

```csharp
public class BlockMyDoor : BlockCompositeTileEntity
{
    public override BlockActivationCommand[] GetBlockActivationCommands(...)
    {
        // Base array belongs to the tile entity — append into a copy.
        BlockActivationCommand[] baseCmds = base.GetBlockActivationCommands(...);
        BlockActivationCommand[] cmds = new BlockActivationCommand[baseCmds.Length + 1];
        Array.Copy(baseCmds, cmds, baseCmds.Length);
        cmds[baseCmds.Length] = new BlockActivationCommand("my_command", "tool", true);
        return cmds;
    }

    public override bool OnBlockActivated(string _commandName, ...)
    {
        if (_commandName == "my_command") { ...; return true; }
        return base.OnBlockActivated(_commandName, ...); // vanilla open/close/lock
    }
}
```

XML: `Extends` the vanilla door and override only `Class` (+ icon/desc):

```xml
<block name="myAutoDoor">
  <property name="Extends" value="elevatorDoor"/>
  <property name="Class" value="BlockMyDoor, MyMod"/>
  <property name="CustomIcon" value="elevatorDoor"/>
</block>
```

The `CompositeFeatures` tree is inherited through `Extends`. The hold-E radial
appears automatically (doors already expose 2+ enabled commands); localize your
command as `blockcommand_my_command` (see Blocks.md → Localization).
