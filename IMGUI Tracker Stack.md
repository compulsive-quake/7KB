# IMGUI Tracker Stack

A reusable in-game HUD pattern for **right-side stacked status cards** that track an arbitrary list of live world entities. Each card shows a name, a horizontal bar (HP / cooldown / progress), and a one-line status string. Cards fade in when an entity enters tracking range, fade out when it dies or leaves. Up to N visible at once.

The pattern uses Unity's **legacy IMGUI** (`OnGUI` on a MonoBehaviour), **not** 7DTD's XUi system.

## Why IMGUI, not XUi

XUi windows that a mod registers in its own `Config/XUi/xui.xml` will not auto-render at game start. Vanilla XUi windows that show up in the world UI tree are members of specific `window_group`s the game opens during HUD setup (compass, hotbar, stat bars, etc.). To get a custom window into that pipeline you'd need either an XPath patch into a vanilla window_group or a Harmony hook on the HUD setup — both fragile.

IMGUI is the cheap escape hatch:

- A `MonoBehaviour` with an `OnGUI()` method draws into the same screen overlay every frame, no XUi registration required.
- Style is bog-standard `GUI.DrawTexture` + `GUI.Label` — no atlases, sliced sprites, or controllers.
- Lifetime is anchored on a `DontDestroyOnLoad` GameObject created in `IModApi.InitMod` — survives world load / death / respawn.

The Scepters mod's `EnchantmentHud.cs` was the original example (world-space floating bars over each enchanted zombie). The Bandits mod's `BanditHud.cs` adapts the same pattern to the right-side stacked layout described here.

## Reference implementations

- [`bandits/src/BanditHud.cs`](../bandits/src/BanditHud.cs) — right-side tracker stack for live bandits in sight range.
- [`Scepters/src/EnchantmentHud.cs`](../Scepters/src/EnchantmentHud.cs) — world-space floating bars over enchanted zombies (different positioning, same draw stack).

## Structure

```csharp
[Preserve]
public class MyTrackerHud : MonoBehaviour
{
    const float TrackRange   = 40f;
    const int   MaxRows      = 5;
    const float PanelWidth   = 240f;
    const float PanelHeight  = 78f;
    const float PanelSpacing = 8f;
    const float RightMargin  = 12f;
    const float FadeIn       = 0.25f;
    const float FadeOut      = 0.6f;

    class Row {
        public int    EntityId;
        public string Name, Status;
        public float  Hp, MaxHp;
        public float  Alpha;
        public bool   Dying;
    }

    readonly Dictionary<int, Row> _rows = new();
    static readonly Dictionary<int, Texture2D> _solidTex = new();
    GUIStyle _nameStyle, _statusStyle;

    void Update()
    {
        SyncFromWorld();   // mark seen entities, flag missing ones as Dying
        TickAlpha();       // step Alpha toward target each frame
    }

    void OnGUI()
    {
        // Iterate rows in stable order, draw a card per row at the right edge.
        // Card y starts at (Screen.height - totalHeight) / 2 to vertically center.
    }
}
```

## Lifecycle / wiring

In your mod's `IModApi.InitMod`:

```csharp
var go = new UnityEngine.GameObject("MyTrackerHud");
UnityEngine.Object.DontDestroyOnLoad(go);
go.AddComponent<MyTrackerHud>();
```

That's it. No XUi registration, no Harmony patches needed.

## SyncFromWorld pattern

Each frame, walk the entity list, pick the candidates that should appear (by type filter + range / state), populate the row dict, and flag any pre-existing row whose entity didn't show this tick as `Dying`. The fade-out handler in `TickAlpha` takes care of removal:

```csharp
var seen = new HashSet<int>();
foreach (var e in world.Entities.list)
{
    if (!(e is MyTargetEntity x) || x.IsDead()) continue;
    float dSq = (x.position - player.position).sqrMagnitude;
    if (dSq > TrackRange * TrackRange) continue;
    seen.Add(x.entityId);

    if (!_rows.TryGetValue(x.entityId, out var row))
        _rows[x.entityId] = row = new Row { EntityId = x.entityId, Alpha = 0f };

    row.Name   = MyDisplayName(x);
    row.Status = x.GetStatusString();
    row.Hp     = x.Health;
    row.MaxHp  = Mathf.Max(1f, x.GetMaxHealth());
    row.Dying  = false;
}
foreach (var kv in _rows)
    if (!seen.Contains(kv.Key)) kv.Value.Dying = true;
```

If you need a max visible count, sort the candidates by distance and only mark the first `MaxRows` as seen — the rest fade out naturally.

## TickAlpha

```csharp
float dt = Time.deltaTime;
List<int> remove = null;
foreach (var kv in _rows)
{
    var r = kv.Value;
    if (r.Dying)
    {
        r.Alpha -= dt / FadeOut;
        if (r.Alpha <= 0f) (remove ??= new()).Add(kv.Key);
    }
    else if (r.Alpha < 1f)
    {
        r.Alpha = Mathf.Min(1f, r.Alpha + dt / FadeIn);
    }
}
if (remove != null)
    foreach (var id in remove) _rows.Remove(id);
```

## OnGUI layout

Stable ordering matters — sort by `EntityId` rather than distance so cards don't shuffle every tick (which would flicker the closest as something moves):

```csharp
var ordered = new List<Row>(_rows.Values);
ordered.Sort((a, b) => a.EntityId.CompareTo(b.EntityId));

float total = ordered.Count * PanelHeight + (ordered.Count - 1) * PanelSpacing;
float y0    = (Screen.height - total) * 0.5f;
float x     = Screen.width - PanelWidth - RightMargin;

for (int i = 0; i < ordered.Count; i++)
{
    var r = ordered[i];
    if (r.Alpha <= 0.001f) continue;
    DrawCard(new Rect(x, y0 + i * (PanelHeight + PanelSpacing), PanelWidth, PanelHeight), r);
}
```

Cards consist of:

1. A semi-opaque dark backing (`GUI.DrawTexture` with a 1×1 solid texture).
2. A 3 px accent strip on the left edge (theme color — red for hostile, green for ally, etc.).
3. A name label (top, bold).
4. An HP bar (mid) — backed dark rect + fill rect + 1px border. Color grades red→yellow→green by `pct`.
5. A status label (bottom, smaller, slightly cool tinted).

Multiply every alpha by `r.Alpha` so the whole card cross-fades on enter/exit.

## Solid 1×1 texture cache

`GUI.DrawTexture` needs a Texture2D, not a Color. Cache one per RGBA tuple:

```csharp
static Texture2D Solid(Color c)
{
    int key = ((int)(c.r * 255) << 24)
            | ((int)(c.g * 255) << 16)
            | ((int)(c.b * 255) << 8)
            |  (int)(c.a * 255);
    if (_solidTex.TryGetValue(key, out var t) && t != null) return t;
    t = new Texture2D(1, 1, TextureFormat.RGBA32, false);
    t.SetPixel(0, 0, c);
    t.Apply();
    return _solidTex[key] = t;
}
```

The cache is keyed on the packed RGBA so identical colors share one texture across all card draws and across the mod lifetime.

## Naming / Theme

For consistency across mods that adopt the pattern, match these conventions:

- File: `<ModName>Hud.cs` (e.g. `BanditHud`, `EnchantmentHud`).
- GameObject name: `<ModName>Hud`.
- Card width 240 px, height 78 px, spacing 8 px, right margin 12 px.
- Backing: `Color(0.08, 0.08, 0.12, 0.82 * alpha)`.
- Accent strip width 3 px, color picked per-mod (Bandits = red, Soul Scepter = purple, etc.).
- Name fontSize 14, bold, white. Status fontSize 12, tinted `Color(0.78, 0.85, 0.95)`.
- HP bar height 14 px, red→yellow→green grade by HP percent.

Anything beyond that (extra rows, icons, animated borders) is up to the mod.

## Gotchas

- **Don't sort visible rows by distance** — the closest card swapping every frame as a target moves looks jittery. Sort by entity id (or pin order on first appearance).
- **Camera reference is optional**. If you're drawing screen-space anchored to `Screen.width / Screen.height`, you don't need `Camera.main`. Only fetch the camera if you're projecting world-space positions like Scepters' `EnchantmentHud` does.
- **`OnGUI` runs on the client only** — don't gate on `IsServer`/`IsDedicatedServer`; the dedicated server has no Screen and won't call OnGUI anyway.
- **One MonoBehaviour per Hud** — don't try to share one OnGUI across multiple unrelated tracker stacks; let each mod own its own HUD GameObject.
- **Texture leaks across reloads** — the `_solidTex` cache is `static`, so its textures survive even if the GameObject is destroyed. That's fine for a small palette (~10 colors) but don't store per-entity textures in it.
