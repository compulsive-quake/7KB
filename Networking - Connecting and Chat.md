# Networking - Connecting and Chat

Part of the [7DTD Modding Knowledgebase](README.md). Covers the client-side networking API: how to connect to a dedicated server programmatically, how to send in-game chat, and the JSON escape gotcha in the [7debug - Remote Debug Server](7debug%20-%20Remote%20Debug%20Server.md) command endpoint.

---

## `ConnectionManager`

`ConnectionManager` is the client-side class responsible for connecting to, disconnecting from, and tracking the current dedicated-server connection. Runtime reflection on `2.6 b14` gives this public instance API:

```csharp
void Connect(GameServerInfo _gameServerInfo);
void openConnectProgressWindow(GameServerInfo _gameServerInfo);
void SetConnectionToServer(INetConnection[] _cons);
INetConnection[] GetConnectionToServer();
void Disconnect();
void DisconnectFromServer();
void DisconnectClient(ClientInfo _cInfo, bool _bShutdown, bool _clientDisconnect);
void Net_ConnectionFailed(string _message);
void Net_DisconnectedFromServer(string _reason);
void Net_PlayerConnected(ClientInfo _cInfo);
void Net_PlayerDisconnected(ClientInfo _cInfo);

// Events
event Action OnDisconnectFromServer;
event ClientConnectionAction OnClientDisconnected;
```

`Connect` is what you want for programmatic join-a-server. `openConnectProgressWindow` is a higher-level wrapper the in-game "Connect to IP" menu uses - it pops the "Connecting to server ..." dialog and then calls `Connect` internally. Both take a single `GameServerInfo` argument containing IP, port, password, and display metadata.

### Singleton instance

**Gotcha:** `ConnectionManager.Instance` via a naive `GetProperty("Instance", Static)` scan does NOT return the live instance. Neither does walking the base type chain looking for a `SingletonMonoBehaviour<T>.Instance` property - that walk returns null at runtime because the base type's `Instance` getter checks a `_instance` field that isn't populated by the usual `Awake()` path on this singleton.

The correct way to reach the live `ConnectionManager` is via `GameManager.Instance.ConnectionManager` (an instance property on `GameManager`, NOT a static on `ConnectionManager`). Or: find any `MonoBehaviour` whose type is `ConnectionManager` via `UnityEngine.Object.FindObjectOfType<ConnectionManager>()`.

If you dump `ConnectionManager`'s methods via reflection you'll successfully see the full API (the type exists), but calling them fails with NullReferenceException or a "no instance" warning unless you grab the instance via the `GameManager` indirection.

### Calling `Connect` from a mod

**Use `Connect`, not `openConnectProgressWindow`.** Both take a single `GameServerInfo` argument and both eventually hit the same network code. The difference is that `openConnectProgressWindow` pops the XUi "Connecting to server..." dialog and then calls `Connect` internally. In `-batchmode -nographics` there is no UI thread driving that window, so the call returns without exception but silently hangs — the handshake never fires. `Connect` is the direct path and works headless.

### Building a `GameServerInfo` — the positional constructors are a trap

`GameServerInfo` is **not** a POJO with `IP`/`Port`/`Password` properties. It is a key-value store with three private dictionaries, and the positional constructors do not map to what their parameter types suggest. On 2.6 b14 the public constructors are:

```csharp
GameServerInfo()                          // parameterless — the one you want
GameServerInfo(GameServerInfo _gsi)       // copy ctor
GameServerInfo(string _serverInfoString)  // parses a serialized form
```

There is **no** `(string ip, int port, string password)` overload despite what naive reflection scans might suggest. Calling any positional ctor with synthetic (host, port, password) arguments produces a `GameServerInfo` with empty IP and `Port = -1`, and the network layer then logs `Connecting to server :-1...` before the handshake dies.

The backing storage is:

```
Dictionary<GameInfoString, string> tableStrings
Dictionary<GameInfoInt,    int>    tableInts
Dictionary<GameInfoBool,   bool>   tableBools
```

keyed by three sibling enums in the same assembly:

- `GameInfoString`: `LevelName, GameMode, IP, SteamID, ServerVersion, Platform, ServerLoginConfirmationText, Region, Language, UniqueId, CombinedPrimaryId, CombinedNativeId, PlayGroup`
- `GameInfoInt`: `Port, CurrentPlayers, MaxPlayers, FreePlayerSlots, GameDifficulty, ...`
- `GameInfoBool`: `IsDedicated, IsPasswordProtected, EACEnabled, AllowCrossplay, ...`

**Note:** there is no `Password` key in `GameInfoString`. Password goes through `GamePrefs` (or separate channels); the `GameServerInfo` only carries `IsPasswordProtected` as a flag.

### Working recipe (verified on 2.6 b14 in a headless client)

```csharp
var cm = GameManager.Instance?.ConnectionManager;  // NOT ConnectionManager.Instance
if (cm == null) return;

// Parameterless ctor, then write directly into the private dicts.
var gsi = new GameServerInfo();

// These fields are non-public, so use reflection. Writing through the
// non-generic IDictionary interface sidesteps the generic type params.
void SetDict(string fieldName, object key, object value)
{
    var f = typeof(GameServerInfo).GetField(
        fieldName, BindingFlags.NonPublic | BindingFlags.Instance);
    var dict = (IDictionary)f.GetValue(gsi);
    dict[key] = value;
}

SetDict("tableStrings", GameInfoString.IP,       "192.168.0.80");
SetDict("tableInts",    GameInfoInt.Port,        26900);
SetDict("tableBools",   GameInfoBool.IsDedicated, true);

// Clear caches so derived getters recompute from the tables.
typeof(GameServerInfo)
    .GetField("cachedToString", BindingFlags.NonPublic | BindingFlags.Instance)
    ?.SetValue(gsi, null);
typeof(GameServerInfo)
    .GetField("isBroken", BindingFlags.NonPublic | BindingFlags.Instance)
    ?.SetValue(gsi, false);

// At this point gsi.IsValid should be true. Fire the connect.
cm.Connect(gsi);
```

After calling `Connect`, the game logs `Connecting to server 192.168.0.80:26900...` and proceeds through the Steam/LiteNetLib handshake normally.

### Readiness — do not call `Connect` too early

Even when `GameManager.Instance` exists and you're at the main menu, the connect call can fail with:

```
EXC The given key 'NetPackageRequestToEnterGame' was not present in the dictionary.
WRN Failed writing first package: NetPackageRequestToEnterGame of size 20.
ERR NCSimple_Serializer Message: The given key 'NetPackageRequestToEnterGame' was not present in the dictionary.
ERR [NET] Kicked from server: Unknown reason
```

`NetPackageManager` registers its package type lookup table late in the boot sequence — empirically around 18-20 seconds after process start on a fast machine. Firing `Connect` before that happens will queue a handshake that cannot serialize its first outgoing package.

Three gates to check before calling `Connect`:

1. `GameManager.Instance != null`
2. `GameManager.Instance.World == null` (you're not already in a game)
3. **`NetPackageManager.GetPackage<NetPackageRequestToEnterGame>() != null`** — the direct readiness probe. This method returns a fresh pooled instance of the requested package type. If the type isn't registered yet, it throws or returns null. Use reflection so you don't have to hard-reference `NetPackageRequestToEnterGame`:

```csharp
static bool IsNetPackageSystemReady()
{
    try
    {
        var pkgType = FindTypeByShortName("NetPackageRequestToEnterGame");
        var nmType  = FindTypeByShortName("NetPackageManager");
        var getPkg  = nmType.GetMethod("GetPackage", BindingFlags.Public | BindingFlags.Static);
        var generic = getPkg.MakeGenericMethod(pkgType);
        return generic.Invoke(null, null) != null;
    }
    catch { return false; }
}
```

**Do not gate on the Unity scene name.** 7DTD 2.6 uses a single scene called `SceneGame` for both the main menu and gameplay — there is no `MainMenu` scene to wait for. The active scene is `SceneGame` from boot onwards.

### Reaching `ConnectionManager` — a gotcha that costs an afternoon

`ConnectionManager.Instance` via a naive `GetProperty("Instance", Static)` scan does NOT return the live instance. Neither does walking the base type chain looking for a `SingletonMonoBehaviour<T>.Instance` property — that walk returns null at runtime because the base type's `Instance` getter checks a `_instance` field that isn't populated by the usual `Awake()` path on this singleton.

The correct way to reach the live `ConnectionManager` is via **`GameManager.Instance.ConnectionManager`** — an instance property on `GameManager`, NOT a static on `ConnectionManager`. Alternatively, `UnityEngine.Object.FindObjectOfType<ConnectionManager>()` also works as a fallback.

If you dump `ConnectionManager`'s methods via reflection you'll successfully see the full API (the type exists), but calling them fails with NullReferenceException or a "no instance" warning unless you grab the instance via the `GameManager` indirection.

---

## Clicking through the spawn-point selection window

After a successful connect, the client loads the world and then presents the "Select spawn point" XUi window. The player has to click one of `NewRandomSpawn` / `NearDeath` / `OnBedRoll` / `NearBedroll` / `NearBackpack` / `NearFriend` / `Unstuck` to actually spawn into the world. For a headless bot you need to drive this programmatically.

### `World != null` is NOT "in game"

A common footgun: checking `GameManager.Instance?.World != null` to decide whether the player is in-game. `World` becomes non-null as soon as the client **starts loading** the world, which is well before the spawn-selection window appears, which is before any player entity exists. If you poll on `World != null` you'll start issuing gameplay commands to a client that is still sitting on the spawn-point chooser.

The accurate in-game signal is:

```csharp
var world = GameManager.Instance?.World;
var primary = world?.GetPrimaryPlayer();
bool inGame = primary != null && !primary.IsDead();
bool atSpawnSelect = world != null && primary == null;
```

### The spawn-selection window and its button

The controller is **`XUiC_SpawnSelectionWindow`**. Its relevant API (2.6 b14):

```csharp
void OnOpen();
void SpawnButtonPressed(SpawnMethod _method, int _nearEntityId);
void showSpawningComponents(ESpawnWindowMode _windowMode);
static bool IsInSpawnSelection(LocalPlayerUI _playerUi);
```

`SpawnButtonPressed` is exactly the method the Spawn button in the UI calls — invoking it directly runs the same game logic as a human clicking.

### The `SpawnMethod` enum

```csharp
enum SpawnMethod
{
    Invalid,          // sentinel — do not use
    NewRandomSpawn,   // first-time spawn, random point in spawn zone
    NearDeath,        // respawn at death location
    OnBedRoll,        // respawn on bedroll
    NearBedroll,
    NearBackpack,
    NearFriend,
    Unstuck,
}
```

**`Invalid` is a sentinel, not a real spawn method.** If you pick `names[0]` from `Enum.GetNames(typeof(SpawnMethod))` as a default you will get `Invalid` — the game may accept it inconsistently but it is not the right choice. For a first-time spawn, use `NewRandomSpawn`.

### Reaching the window from a mod

`XUiC_SpawnSelectionWindow` is an XUi *controller*, not a `MonoBehaviour`, so `FindObjectOfType` won't find it. It lives inside the XUi tree hanging off a `LocalPlayerUI`. The walk is:

```csharp
// 1. Find any live LocalPlayerUI (there's one per local player).
var lpus = (LocalPlayerUI[])UnityEngine.Object.FindObjectsOfType(typeof(LocalPlayerUI));

foreach (var lpu in lpus)
{
    var xui = lpu.xui;                       // XUi instance
    var window = xui.GetChildByType<XUiC_SpawnSelectionWindow>();
    if (window == null) continue;

    window.SpawnButtonPressed(SpawnMethod.NewRandomSpawn, -1);
    break;
}
```

`-1` for `_nearEntityId` means "no reference entity" (which is what `NewRandomSpawn` expects anyway).

### Alternate entry points (useful to know but not always sufficient)

`GameManager` exposes a few spawn-related methods that look like shortcuts but are not all equivalent to the UI button:

```csharp
void DoSpawn();
void RequestToSpawn(int _nearEntityId);                                   // direct request
void RequestToSpawnPlayer(ClientInfo _cInfo, int _chunkViewDim,
                          PlayerProfile _playerProfile, int _nearEntityId); // server-side
```

`GameManager.RequestToSpawn(-1)` is the closest to the UI button. It may work as a fallback if the `XUiC_SpawnSelectionWindow` walk fails (e.g. if the window hasn't been constructed yet). `RequestToSpawnPlayer` is the server-side path and needs a `ClientInfo` you probably don't have on the client side.

---

## Empirical timeline — cold launch to in-game (headless 2.6 b14)

| t (s) | State | Notes |
|---|---|---|
| 0 | process starts | launch-bot.ps1 / Start-Process exits |
| ~6 | mod HTTP API up | `GameManager.Instance` is NOT yet alive |
| ~13 | `GameManager.Instance != null` | but `NetPackageManager` still registering |
| ~20 | `NetPackageManager` ready | **earliest safe moment to call `Connect`** |
| ~30 | `World != null`, spawn-select window visible | call `SpawnButtonPressed(NewRandomSpawn, -1)` |
| ~48 | primary player entity exists | truly in-game |

The ~20-second floor comes from `NetPackageManager` initialization; the ~30-second world-load is network/disk bound (faster for small save files, slower for big ones). The 18-second gap between spawn-select appearing and the player entity actually spawning is chunk streaming and player construction.

---

## `GameManager.ChatMessageClient` vs `GameManager.ChatMessageServer`

Both methods exist on `GameManager` and both take chat text, but they mean different things:

```csharp
// Client-side: display a chat message in MY OWN chat window only.
// Does NOT broadcast to other clients. Used by the receive-side chat
// handler when the server pushes a chat packet to this client.
public void ChatMessageClient(
    EChatType _chatType,
    int _senderEntityId,
    string _msg,
    List<int> _recipientEntityIds,
    EMessageSender _msgSender,
    BbCodeSupportMode _bbMode);

// Server-side only: broadcast a chat message to the specified clients
// (or all clients). Calling this from a client machine will NOT send
// the message over the wire - the server is the only thing that can
// authoritatively broadcast chat. From a client, it silently no-ops
// or displays locally depending on the code path.
public void ChatMessageServer(
    ClientInfo _cInfo,
    EChatType _chatType,
    int _senderEntityId,
    string _msg,
    List<int> _recipientEntityIds,
    EMessageSender _msgSender,
    BbCodeSupportMode _bbMode);
```

**Don't** try to call `ChatMessageServer` from a client mod to broadcast chat - it won't reach other players. The correct path is to construct a `NetPackageChat` and send it up to the server, which then fans it out via its own `ChatMessageServer`.

### Sending chat as the local player

From a client mod, the idiomatic way to say something in-game as the local player:

```csharp
var pkg = NetPackageManager.GetPackage<NetPackageChat>().Setup(
    EChatType.Global,
    player.entityId,    // sender
    text,               // message
    null,               // recipient entity IDs (null = all)
    EMessageSender.Player);
SingletonMonoBehaviour<ConnectionManager>.Instance.SendToServer(pkg);
```

The `SendToServer` call hands the packet to the server, which then broadcasts it as `GMSG: Player '<name>' said "..."` and every client (including the sender) displays it in their chat window with the player's name.

### The `say` console command is NOT this

`say` is a server-only console command. On a dedicated server, it broadcasts `Server: <text>`. On a non-dedicated client, it used to broadcast nothing at all in older versions but in 2.6 the behavior is ambiguous - the author observed `say "..."` from a bot client broadcasting as either `Server:` or not at all, depending on state. **If you want player-originated chat from a mod, construct a `NetPackageChat` via the method above.** Don't rely on `say`.

---

## JSON escape gotcha in 7debug's `/api/command` endpoint

This is a bug that shipped in the 7debug mod from its first release and manifests as **chat messages broadcasting as a literal `\` (backslash) character** when the command contains a double-quoted substring.

### Symptom

Posting `POST /api/command` with body:

```json
{"command":"say \"hello\""}
```

results in the chat window showing `Server: \` instead of `Server: hello`.

### Cause

The mod's JSON parser (`DebugHttpServer.ExtractJsonString`) walks the string value looking for the closing quote and correctly skips over `\X` escape sequences so the closing quote detection works. **But it then returns the raw substring without unescaping the `\X` sequences back to their literal characters.** So the game console receives `say \"hello\"` with literal backslashes, parses it as "the message body starts at `\`", and the quote consumption logic collapses it down to nothing useful.

### Fix

`ExtractJsonString` needs to build the decoded string char-by-char, translating:
- `\"` → `"`
- `\\` → `\`
- `\n` → newline, `\r` → CR, `\t` → tab, etc.
- `\uXXXX` → the Unicode codepoint

Any 7debug-derived mod that handles chat (or any command containing quoted substrings) needs this fix. See `mod/src/DebugHttpServer.cs` in the ClaudePlays7Days project for a reference implementation.

### Workaround without fixing the mod

If you can't or don't want to patch the mod, avoid double quotes in the command payload. For `say`, that means pre-stripping `"` from the text before wrapping it in `say "..."`. Less robust but it avoids the bug entirely.

---

## Related

- [7debug - Remote Debug Server](7debug%20-%20Remote%20Debug%20Server.md) - HTTP API surface that exposes these commands remotely
- [Harmony Patching](Harmony%20Patching.md) - how to patch server-side authorizers and client-side UI controllers
- [Entity Positions](Entity%20Positions.md) - reading player positions, the companion read-side to this write-side doc
