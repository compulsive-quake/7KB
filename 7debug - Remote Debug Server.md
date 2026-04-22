# 7debug - Remote Debug HTTP Server

The `7debug` mod runs an HTTP server on **port 7860** inside the game process. Any mod, tool, or AI agent can connect to it to inspect game state, execute commands, capture screenshots, and read logs — without needing to be in the same process.

## Connecting

Base URL: `http://localhost:7860`

All responses are JSON (except screenshots which return `image/png`). CORS is enabled for browser access.

## Endpoints

### GET /api/status
Game status and performance metrics.
```json
{
  "inGame": true,
  "fps": 60.0,
  "memoryMB": 1024.5,
  "unityMemoryMB": 2048.3,
  "uptime": "01:23:45",
  "platform": "WindowsPlayer",
  "gameVersion": "...",
  "gameTime": {"day": 7, "hour": 14}
}
```

### GET /api/players
All connected players with position, health, stamina, level.
```json
{
  "players": [
    {
      "entityId": 171,
      "name": "PlayerName",
      "position": {"x": 100.0, "y": 50.0, "z": -200.0},
      "rotation": {"x": 0.0, "y": 180.0, "z": 0.0},
      "health": 100,
      "maxHealth": 150,
      "stamina": 100,
      "level": 25,
      "isDead": false
    }
  ]
}
```

Position/rotation are live world coordinates from `EntityAlive.position`. See [Entity Positions](Entity%20Positions.md) for axis conventions, sanity ranges, and the origin-reposition behavior that affects Unity transforms but not this API.

### GET /api/world
World info: name, seed, size, game day/hour, difficulty, entity and chunk counts.
```json
{
  "worldName": "Navezgane",
  "seed": "myseed",
  "worldSize": 6144,
  "gameTime": {"day": 7, "hour": 14},
  "difficulty": 3,
  "entityCount": 42,
  "chunkCacheCount": 256
}
```

### GET /api/entities
Up to 200 active entities with type, name, position, health, alive status.
```json
{
  "entities": [
    {
      "entityId": 500,
      "type": "EntityZombie",
      "name": "zombieBoe",
      "position": {"x": 105.0, "y": 48.0, "z": -195.0},
      "health": 150,
      "isDead": false
    }
  ]
}
```

### GET /api/mods
All loaded mods with name, display name, version, and file path.
```json
{
  "mods": [
    {"name": "7debug", "displayName": "7Debug - Remote Debug HTTP Server", "version": "1.0.0", "path": "..."}
  ]
}
```

### GET /api/console
Last 100 log messages. Errors and exceptions include stack traces.
```json
{
  "log": [
    {"time": "14:30:05.123", "type": "Log", "message": "Something happened"},
    {"time": "14:30:06.456", "type": "Error", "message": "NullRef...\n  at Foo.Bar()..."}
  ]
}
```

### GET /api/screenshot
Captures the current frame and returns it as `image/png`. Only works on client (not dedicated server).

Use this for:
- Automated visual bug detection
- AI-assisted debugging (send screenshot to vision model)
- Regression testing of visual changes

### POST /api/command
Execute any console command. Send JSON body:
```json
{"command": "listplayers"}
```
Response includes captured log output:
```json
{
  "command": "listplayers",
  "output": ["1 players connected:", "0. id=171, ..."]
}
```

### POST /api/reloadgame
Exits the current game, waits for the main menu, then reloads the same save. Picks up all XML changes (XUi, blocks, items, recipes, localization) without restarting the game process. Does NOT reload C# DLLs — those require a full restart.

Send an empty body `{}`.
```json
{"status": "reloading", "world": "Juvupe Valley", "game": "derp"}
```
**Note:** Only works when already in a game (`inGame: true`). Use `/api/loadgame` if on the main menu.

### GET /api/saves
List all saved games, sorted by most recently modified.
```json
{
  "saves": [
    {"world": "Juvupe Valley", "game": "derp", "lastModified": "2026-03-25 00:30:13"},
    {"world": "Pregen06k02", "game": "My Game", "lastModified": "2026-03-20 14:00:00"}
  ]
}
```

### POST /api/loadgame
Load a saved game. Send an empty body `{}` to load the most recent save, or specify world and game:
```json
{"world": "Juvupe Valley", "game": "derp"}
```
Response:
```json
{"status": "loading", "world": "Juvupe Valley", "game": "derp"}
```
**Note:** Only works from the main menu (`inGame: false`). Returns an error if already in a game.

Common useful commands:
- `listplayers` — list connected players
- `gettime` — current game time
- `settempunit f/c` — set temperature unit
- `debugmenu` — toggle debug menu
- `spawnairdrop` — spawn an airdrop
- `spawnentity <id> <x> <y> <z>` — spawn entity at position
- `give <item> <count>` — give items to player
- `teleport <x> <y> <z>` — teleport player
- **`xui reload`** — re-parse every XUi window group from disk (layouts, styles, localization, custom atlas sprites) without restarting the game. Primary tool for fast XML-edit iteration from an AI agent; see the XUi Window System doc's *Hot-Reloading XUi* section for what does and doesn't reload.

### Iterating on XUi edits without a restart

```bash
# 1. Sync the changed file(s) into the deployed mod folder
cp MyMod/Config/XUi/windows.xml "C:/Program Files (x86)/Steam/steamapps/common/7 Days To Die/Mods/MyMod/Config/XUi/windows.xml"

# 2. Reload XUi remotely
curl -s -X POST http://localhost:7860/api/command \
  -H "Content-Type: application/json" \
  -d '{"command":"xui reload"}'
# -> {"command":"xui reload","output":[]}

# 3. Verify success in the log (look for "Parsing all window groups completed")
curl -s http://localhost:7860/api/console | jq '.log[-10:] | .[].message'
```

`xui reload` prints its progress via game `INF` logs, not via the captured command buffer — so `output` in the HTTP response is expected to be empty. Check `/api/console` to confirm the reload actually ran.

Full restart is still required when editing mod DLLs (controllers, Harmony patches), `items.xml`, `blocks.xml`, `recipes.xml`, or anything baked during initial game load.

## Usage from other mods / tools

### From C# (another mod in the same game):
```csharp
using System.Net.Http;

var client = new HttpClient { BaseAddress = new Uri("http://localhost:7860") };

// Get game status
var status = await client.GetStringAsync("/api/status");

// Run a command
var content = new StringContent("{\"command\":\"listplayers\"}", Encoding.UTF8, "application/json");
var result = await client.PostAsync("/api/command", content);

// Take a screenshot and save it
var png = await client.GetByteArrayAsync("/api/screenshot");
File.WriteAllBytes("screenshot.png", png);
```

### From Python / shell scripts:
```bash
# Status
curl http://localhost:7860/api/status

# Run command
curl -X POST http://localhost:7860/api/command -d '{"command":"listplayers"}'

# Screenshot
curl http://localhost:7860/api/screenshot -o screenshot.png
```

### From an AI agent (Claude Code, etc.):
```bash
# Check if game is running and get state
curl -s http://localhost:7860/api/status | jq .

# Capture screenshot for visual analysis
curl -s http://localhost:7860/api/screenshot -o /tmp/game.png

# Read recent errors
curl -s http://localhost:7860/api/console | jq '.log[] | select(.type == "Error")'
```

## Notes
- The server starts automatically when the game loads the mod
- Port 7860 is hardcoded (change in `src/ModApi.cs` if needed)
- Screenshots only work on client, not dedicated server
- Entity list is capped at 200 to keep response sizes manageable
- Log buffer holds 500 entries; `/api/console` returns last 100
- The server shuts down cleanly when the game exits
