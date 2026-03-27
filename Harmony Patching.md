# Harmony Patching

Part of the [7DTD Modding Knowledgebase](README.md). Covers Harmony patch patterns for 7DTD mods — initialization, prefix/postfix, reflection, and common targets.

---

## Overview

7DTD mods use HarmonyLib to patch game methods at runtime. Harmony is included with the game (via `0Harmony.dll`). Patches let you intercept, modify, or replace game behavior without touching the game's source code.

---

## Initialization

In your `IModApi.InitMod`:

```csharp
[Preserve]
public class ModApi : IModApi
{
    public void InitMod(Mod _modInstance)
    {
        Harmony harmony = new Harmony("com.myname.mymod");
        harmony.PatchAll(); // patches all [HarmonyPatch] classes in this assembly
        Log.Out("[MyMod] Init complete");
    }
}
```

Use a unique Harmony ID (reverse domain notation) to avoid conflicts with other mods.

---

## Patch Types

### Prefix (runs before the original method)

```csharp
[HarmonyPatch(typeof(TargetClass), "TargetMethod")]
public static class MyPatch
{
    // Return false to skip the original method
    static bool Prefix(TargetClass __instance)
    {
        if (ShouldSkip(__instance))
            return false; // skip original
        return true;      // run original
    }
}
```

### Postfix (runs after the original method)

```csharp
[HarmonyPatch(typeof(TargetClass), "TargetMethod")]
public static class MyPatch
{
    static void Postfix(TargetClass __instance, ref int __result)
    {
        // Modify the return value
        __result = 42;
    }
}
```

### Finalizer (runs regardless of exceptions)

Finalizers run after the original method completes, whether it succeeded or threw an exception. Use them to suppress or transform exceptions. The `__exception` parameter receives the thrown exception (or `null` if none). Return `null` to suppress it, or return the exception to let it propagate.

```csharp
[HarmonyPatch(typeof(TargetClass), "TargetMethod")]
public static class MyPatch
{
    static Exception Finalizer(Exception __exception)
    {
        if (__exception is NullReferenceException)
            return null; // suppress this specific exception type
        return __exception; // let all others propagate
    }
}
```

Key points:
- Always filter by exception type — suppressing all exceptions blindly masks real bugs
- Finalizers run even if a Prefix returned `false` (skipped the original)
- You can inject `__instance`, `__result`, etc. just like Prefix/Postfix
- Use sparingly — prefer fixing the root cause when possible. Finalizers are appropriate when the crash is in game code you can't modify (e.g., vanilla UI controllers that don't handle edge cases)

### Prefix + Postfix with state

Pass state from prefix to postfix using `__state`:

```csharp
static void Prefix(ref ushort power, out ushort __state)
{
    __state = power; // save original value
}

static void Postfix(ref ushort power, ushort __state)
{
    // __state has the value from Prefix
    if (power < __state) { /* power was reduced */ }
}
```

---

## Dynamic Target Methods

When the target method name varies across game versions, use `TargetMethod()`:

```csharp
[HarmonyPatch]
public static class MyPatch
{
    static MethodBase TargetMethod()
    {
        // Try multiple method names
        string[] candidates = { "OnBlockPlaced", "OnBlockAdded", "PlaceBlock" };
        foreach (string name in candidates)
        {
            var mi = AccessTools.Method(typeof(BlockPowerSource), name);
            if (mi != null) return mi;
        }
        throw new MissingMethodException("Could not find target method");
    }

    static void Postfix(object[] __args) { /* ... */ }
}
```

---

## Accessing Private Fields

### Via `__instance` + reflection

```csharp
static bool Prefix(XUiC_PowerSourceWindowGroup __instance)
{
    var field = __instance.GetType().GetField("tileEntity",
        BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
    var te = field?.GetValue(__instance) as TileEntityPowerSource;
}
```

### Via Harmony's `AccessTools`

```csharp
var field = AccessTools.Field(typeof(PowerItem), "Parent");
var parent = field?.GetValue(item) as PowerItem;

var prop = AccessTools.Property(typeof(TileEntity), "PersistentData");
var data = prop?.GetValue(te, null);
```

---

## Common Patch Targets

### Block interaction

| Target | Purpose |
|---|---|
| `BlockPowered.OnBlockActivated` | Intercept E press on powered blocks |
| `BlockPowerSource.updateState` | Runs every tick for power sources (visual updates) |
| `BlockPowerSource.OnBlockPlaced` | Block placement |

### Power system

| Target | Purpose |
|---|---|
| `PowerGenerator.TickPowerGeneration` | Fuel consumption / power generation tick |
| `PowerSource.RefreshPowerStats` | Recalculate MaxOutput, MaxPower |
| `PowerSource.HandleSendPower` | Power distribution to consumers |
| `PowerItem.HandlePowerReceived` | Consumer receiving power |

### UI

| Target | Purpose |
|---|---|
| `XUiC_PowerSourceWindowGroup.OnOpen` | Redirect power source UI to custom window |
| `XUiC_WorkstationWindowGroup.OnOpen` | Redirect workstation UI |
| `XUi.Init` | Inject custom UI atlas entries after UI system initializes |

### TileEntity persistence

| Target | Purpose |
|---|---|
| `TileEntityX.read(PooledBinaryReader)` | Load custom data from save |
| `TileEntityX.write(PooledBinaryWriter)` | Save custom data to save |
| `TileEntityPowerSource.ReplacedBy` | Block destroyed — drop items |

---

## Intercepting UI Window Open

To redirect a vanilla UI to your custom window:

```csharp
[HarmonyPatch(typeof(XUiC_PowerSourceWindowGroup), "OnOpen")]
public class Patch_RedirectUI
{
    static bool Prefix(XUiC_PowerSourceWindowGroup __instance)
    {
        // Check if this is your custom block
        var te = GetTileEntity(__instance);
        if (te == null || !(te is TileEntityMyBlock))
            return true; // let vanilla handle

        // Close vanilla, open custom
        var wm = __instance.xui.playerUI.windowManager;

        // Close vanilla window (use reflection for version safety)
        TryClose(wm, "powersource");

        // Open custom window group
        TryOpen(wm, "MyWindowGroup", modal: true);

        // Assign tile entity to custom controller
        var ctrl = FindController(__instance.xui, "windowMyBlock");
        if (ctrl != null)
            ctrl.TileEntity = te;

        return false; // skip vanilla OnOpen
    }
}
```

---

## GUIWindowManager Reflection

`GUIWindowManager.Open` and `Close` signatures vary by game version. Use reflection:

```csharp
static bool TryOpen(object wm, string key, bool modal)
{
    var method = wm.GetType().GetMethod("Open",
        BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic,
        null,
        new[] { typeof(string), typeof(int), typeof(int), typeof(bool), typeof(bool) },
        null);
    if (method != null)
    {
        method.Invoke(wm, new object[] { key, -1, -1, modal, false });
        return true;
    }
    return false;
}
```

---

## Patching Type.GetType for Mod Controllers

The game's XUi system uses `Type.GetType(name)` to resolve controller classes, which only searches `Assembly-CSharp`. Mod assemblies aren't found. Fix with Harmony patches on `Type.GetType`:

```csharp
[HarmonyPatch(typeof(Type), "GetType", new[] { typeof(string) })]
public static class Patch_TypeGetType
{
    static void Postfix(string __0, ref Type __result)
    {
        if (__result != null) return;

        // Strip assembly qualifier if present (e.g. "MyBlock, MyMod" -> "MyBlock")
        string plainName = __0;
        int commaIdx = __0.IndexOf(',');
        if (commaIdx >= 0)
            plainName = __0.Substring(0, commaIdx).Trim();

        foreach (var asm in AppDomain.CurrentDomain.GetAssemblies())
        {
            // Try full name first (handles namespace-qualified names)
            __result = asm.GetType(plainName);
            if (__result != null) return;

            // Fall back to short name search (handles namespaced classes)
            try
            {
                foreach (var t in asm.GetTypes())
                {
                    if (t.Name == plainName)
                    {
                        __result = t;
                        return;
                    }
                }
            }
            catch { }
        }
    }
}
```

> **Warning:** `Assembly.GetType()` does NOT accept assembly-qualified names like `"MyBlock, MyMod"` — you must strip the assembly part first. Also, if your classes are in a namespace, `asm.GetType("MyBlock")` won't find `MyNamespace.MyBlock` — you need to search by short name as a fallback.

Patch all three overloads: `GetType(string)`, `GetType(string, bool)`, `GetType(string, bool, bool)`.

---

## Gotchas

> **`[Preserve]` attribute** — Add to `IModApi` and block classes. Unity's IL stripping can remove classes only referenced via reflection/XML.

> **Reflection parameter names** — Use positional injection (`string __0`) rather than named parameters in Harmony patches on system methods (e.g., `Type.GetType`). Mono's internal parameter names may differ from the documentation.

> **Server vs client** — Many patches need `if (ConnectionManager.Instance.IsServer)` or `if (GameManager.IsDedicatedServer)` guards. Power generation runs on server; visual effects run on client.

> **`SendHasLocalChangesToRoot()`** — Call this after modifying any `PowerItem` property to propagate changes through the power network.

> **`SetModified()`** — Call on TileEntity after any state change to ensure persistence and client sync.

> **Multiple postfixes on the same method** — Harmony patches from different mods stack. Your postfix on `BlockPowerSource.updateState` will run alongside other mods' patches. Don't assume you're the only one patching.

> **Never close/reopen window groups from inside OnOpen** — Calling `wm.Close("groupA")` then `wm.Open("groupB")` from within a `Prefix` on `XUiC_*WindowGroup.OnOpen` causes NullReferenceExceptions. Closing the group mid-initialization disrupts the tile entity handoff (e.g., `player.lootContainer` gets cleared). Instead, redirect at the `GUIWindowManager.Open` level by patching `GUIWindowManager.Open` and changing the window group name string *before* the window system starts initializing. Use a static flag set in a block activation patch to know when to redirect:
>
> ```csharp
> // 1. Track activation in BlockSecureLoot.OnBlockActivated Prefix
> static void Prefix(BlockValue _blockValue) {
>     RedirectPatch.IsMyBlockOpening = _blockValue.Block?.GetBlockName() == "myBlock";
> }
>
> // 2. Redirect in GUIWindowManager.Open Prefix
> static void Prefix(ref string __0) {
>     if (__0 == "looting" && RedirectPatch.IsMyBlockOpening) {
>         RedirectPatch.IsMyBlockOpening = false;
>         __0 = "myCustomGroup";
>     }
> }
> ```
