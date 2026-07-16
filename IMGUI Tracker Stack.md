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

---

## IMGUI Tool Overlays (sliders, alignment panels)

The tracker pattern above is read-only — pure display. The same `OnGUI` hook is also used for **interactive tool overlays** like alignment panels and live-tuning sliders (e.g. zPhone's `JigsawTuneTool`, `ScepterScaleSliderTool`, RetroZed's `ArcadeAlignTool`). They share the same gotchas plus a few unique to interactive IMGUI.

### Always allow manual number entry next to drag/step controls

Any alignment or tuning panel that exposes a numeric value (scale, rotation, offset, position, color channel, etc.) **must** include a typeable text field, not just `+`/`-`/`<<`/`>>` step buttons or a drag slider. Stepping is fine for "nudge to taste"; typing is the only way to enter an exact value or paste one back from a saved-values dump. Skipping the text field forces the user to spam-click step buttons to reach a known value, which is slow and error-prone.

Layout per row: `<<` (hold-rewind) / `-` / **editable text field** / `+` / `>>` (hold-ffwd). The text field absorbs whatever width is left after the four 30-px buttons.

For a slider row where the readout doubles as the text field (click the number,
type, Enter/click-away commits, Esc cancels), see
`FPV/src/FPVOptionsDialog.cs::NumberField`. Its gotchas:

- Keep an **edit buffer** (`_editField` name + `_editText`) while the field is
  focused and only reformat when unfocused — otherwise `v.ToString(fmt)` every
  frame fights the user (can't type an intermediate `"1."`).
- IMGUI buttons/custom sliders are focus-passive, so clicking them never blurs
  a text field. Blur explicitly: on any `MouseDown` at the top of the draw, set
  `GUIUtility.keyboardControl = 0` — a click ON a field refocuses it in the same
  event, a click elsewhere leaves it blurred and the field commits on focus loss.
- If the panel closes on Esc, check "is a field being edited?" first and make
  Esc cancel the edit instead of closing.
- Don't snap typed values to the slider's step increment — quantize only drags.

```csharp
// One tunable row. Returns next Y.
float DrawRow(float px, float y, float panelW, string label, string fmt,
              float step, string fieldName, ref string text)
{
    GUI.Label(new Rect(px, y, panelW, 22), label, _label);
    y += 22;

    const float bw = 30f, gap = 6f;
    float tw = panelW - (bw * 4 + gap * 4);
    float xRewind = px;
    float xMinus  = xRewind + bw + gap;
    float xText   = xMinus  + bw + gap;
    float xPlus   = xText   + tw + gap;
    float xFfwd   = xPlus   + bw + gap;

    float current = GetFloat(fieldName);

    // <<  hold-rewind: gate on Repaint so we don't fire 200×/sec.
    if (GUI.RepeatButton(new Rect(xRewind, y, bw, 28), "<<", _btn)
        && Event.current.type == EventType.Repaint)
        SetFloat(fieldName, current -= step);

    if (GUI.Button(new Rect(xMinus, y, bw, 28), "-", _btn))
        SetFloat(fieldName, current -= step);

    // Forgiving parse: hold partial input ("", "-", "1.") until it's
    // a complete float distinct from the current value. Without this,
    // typing "1.05" gets clobbered to 1 the moment you type "1".
    if (text == null) text = current.ToString(fmt, CultureInfo.InvariantCulture);
    string newText = GUI.TextField(new Rect(xText, y, tw, 28), text, _txt);
    if (newText != text)
    {
        text = newText;
        string trimmed = newText.Trim();
        if (trimmed.Length > 0 && trimmed != "-" && !trimmed.EndsWith(".") &&
            float.TryParse(trimmed, NumberStyles.Float, CultureInfo.InvariantCulture, out float parsed) &&
            !Mathf.Approximately(parsed, current))
        {
            SetFloat(fieldName, parsed);
        }
    }

    if (GUI.Button(new Rect(xPlus, y, bw, 28), "+", _btn))
        SetFloat(fieldName, current += step);

    if (GUI.RepeatButton(new Rect(xFfwd, y, bw, 28), ">>", _btn)
        && Event.current.type == EventType.Repaint)
        SetFloat(fieldName, current += step);

    return y + 32;
}
```

After +/-/Reset/external mutation, re-sync the local text mirrors so the field reflects the new authoritative value (e.g. `_scaleText = GetFloat("Scale").ToString(fmt, CultureInfo.InvariantCulture)`). Reference: `zPhone/src/apps/Jigsaw/JigsawTuneTool.cs::DrawRow`.

### IMGUI does **not** block parallel `Input.*` readers

`Event.current.Use()` consumes the event for the IMGUI dispatch but does **not** suppress `Input.GetMouseButtonDown(int)` / `Input.GetKeyDown(KeyCode)` reads happening that same frame on other MonoBehaviours. Two systems both see the click.

This bites any mod that has an input listener (zPhone's `ZPhoneKeyListener`, RetroZed's arcade-key hook, etc.) AND an interactive IMGUI tool. Symptoms:

- Click an IMGUI button → the listener also reacts → an unrelated phone/window opens, or fires the listener's primary action a second time.
- Clicking IMGUI close + immediate reopen looks like the panel is "flickering".

**Fix**: each interactive IMGUI tool exposes a static `IsOpen` accessor so other systems can short-circuit while it's up. Critically, the accessor must NOT instantiate the singleton — only read its cached instance:

```csharp
public class JigsawTuneTool : MonoBehaviour
{
    static JigsawTuneTool _instance;
    bool _active;

    // null-safe — does not call Instance, so reading from the listener
    // doesn't spawn the GameObject before the user opens the tool.
    public static bool IsOpen => _instance != null && _instance._active;
}
```

Then any cross-cutting input reader gates on it:

```csharp
private void TryOpenPhone()
{
    if (JigsawTuneTool.IsOpen) return;
    if (ScepterScaleSliderTool.IsOpen) return;
    // … rest of the open logic
}
```

If you add a new interactive IMGUI tool, give it an `IsOpen` accessor and add it to every existing input listener's gate list. Reference: `zPhone/src/ZPhoneKeyListener.cs::TryOpenPhone`.

### Scale fixed-pixel panels to the resolution (GUI.matrix against a 1080 reference)

IMGUI lays out in **raw pixels**, so a panel authored with hardcoded sizes (window `760×600`, `fontSize 14`, `GUILayout.Width(100)`, etc.) renders the same pixel count on every display — which means it looks correctly sized at 1080p but shrinks and gets hard to read at 1440p/4K. Don't hand-scale each width/font; wrap the whole draw in a `GUI.matrix` scale keyed to a 1080p reference (the same `HudReferenceHeight = 1080f` convention the HUD uses), and lay everything out in reference pixels:

```csharp
float scale = Screen.height / 1080f;          // 1.0 @1080p, 1.333 @1440p, 2.0 @4K
Matrix4x4 prev = GUI.matrix;
GUI.matrix = Matrix4x4.Scale(new Vector3(scale, scale, 1f));

float vw = Screen.width / scale, vh = 1080f;   // screen size in reference px
// ... lay out the panel centered in (vw × vh), full-screen dim over (0,0,vw,vh) ...

GUI.matrix = prev;                             // restore so scale doesn't leak
```

- At 1080p `scale == 1`, so the layout is byte-for-byte unchanged — safe to retrofit onto a panel that already "looks good at 1080p".
- The matrix applies to **event coordinates too**, so `GUILayout.Button`/`Toggle` hit-testing stays correct; no manual mouse-coord fixups. (Manual `Input.mousePosition` hit-tests would need dividing by `scale`.)
- Always restore `GUI.matrix` at the end of the draw, or the scale bleeds into any other `OnGUI` drawn afterward.
- Text is bitmap-scaled by the matrix, so it's slightly soft at large factors (2×) — fine for a settings/tool panel; if you need crisp glyphs, scale `fontSize` by `scale` instead and skip the matrix (but then every `Width`/`Rect` needs `* scale` too).

Reference: `FPV/src/FPVOptionsDialog.cs::Draw`.

### Shrunk in-panel preview of a full-screen HUD (don't nest GUI.matrix in a layout area)

To show a live miniature of a full-screen IMGUI HUD inside a settings panel (e.g. per-element
visibility toggles next to a preview), do **not** try to compose a second `GUI.matrix`
scale inside a `GUILayout.BeginArea`/`BeginScrollView` — the clip stack's offsets were
computed under the outer matrix, and changing the matrix mid-area scales those offsets
too, landing the preview in the wrong place.

Instead, factor the HUD's element drawing into a shared static renderer that draws in the
HUD's reference canvas (1920×1080) and routes every rect through an explicit
origin+scale transform (identity for the live feed path):

```csharp
static float _ox, _oy, _scale = 1f;
static Rect R(float x, float y, float w, float h)
{
    if (_scale == 1f) return new Rect(x, y, w, h);
    return new Rect(_ox + x * _scale, _oy + y * _scale,
        Mathf.Max(1f, w * _scale), Mathf.Max(1f, h * _scale));  // 1px floor: see below
}
```

- The **1px floor** on scaled sizes matters: a 2px HUD stroke at preview scale (~0.3×)
  is 0.6px and vanishes/dashes without it.
- Scale `fontSize` by the same factor (floor ~6px) when building the preview's styles;
  the live path keeps full-size fonts.
- The settings dialog gets the preview rect from `GUILayoutUtility.GetRect(w, h)` and
  passes `rect.x, rect.y, rect.height / 1080f` as the transform — works inside scroll
  views because everything stays in plain layout coordinates.
- Sharing one renderer between feed and preview (live values vs. a fixed sample struct)
  keeps the preview from drifting when the HUD layout changes.

Reference: `FPV/src/FPVHudRenderer.cs` (`DrawScaled`) + `FPV/src/FPVOptionsDialog.cs::DrawHudTab`.

### Don't rely on the game skin's default control styles (tiny fonts, invisible sliders)

7DTD's `GUI.skin` renders IMGUI defaults *small*: ~11-12 px label/button text, and
`GUILayout.HorizontalSlider` draws as a hairline track with a barely-visible thumb —
users can't even tell it's a slider. For any user-facing panel:

- Build explicit `GUIStyle`s once (`new GUIStyle(GUI.skin.button) { fontSize = 16, padding = ... }`)
  and pass them to **every** `Button`/`Toggle`/`Label` call; don't mutate `GUI.skin` in place
  (it's shared with the rest of the game's IMGUI).
- Give row labels a `fixedHeight` matching the button height + `TextAnchor.MiddleLeft`
  so text centres against taller buttons instead of riding the row top.
- Replace `HorizontalSlider` with a custom-drawn slider: `GUILayoutUtility.GetRect` +
  `GUIUtility.GetControlID(hash, FocusType.Passive, rect)` + `hotControl` mouse handling
  (MouseDown in rect → grab, MouseDrag while hot → set value from mouse X, MouseUp → release,
  draw track/fill/thumb on `EventType.Repaint` only). Flank it with `−`/`+` step buttons for
  precision. Reference: `FPV/src/FPVOptionsDialog.cs::FancySlider` / `SteppedSlider`.
- Unicode glyphs proven to render in the game font (used in FPV): `▶ − + × …`. Fancier
  glyphs (⚙ ↺ ⌨) are unverified and may render as boxes — test before shipping.

### Layering two IMGUI scripts (GUI.depth)

When two separate MonoBehaviours both draw in `OnGUI` (e.g. a modal dialog with a
full-screen dim + a HUD widget that should stay visible on top of the dim), their
relative draw order is undefined unless you set `GUI.depth` in each. **Lower depth
renders on top.** `GUI.depth` is a static shared across all OnGUI scripts, so pin it
explicitly in *both* scripts' `OnGUI` (e.g. dialog `GUI.depth = 0`, widget
`GUI.depth = -10` while the dialog is open) — otherwise one script's value leaks into
the other and the ordering flips unpredictably. Used by Uncover's
`MinimapWidget`/`MinimapSettingsDialog` so the live minimap preview draws above the
dialog's 0.55-alpha dim.

### Other interactive-IMGUI gotchas

- **`Cursor.visible = true; Cursor.lockState = CursorLockMode.None`** in `Update` while active — the game re-locks the cursor every frame, so set them once in `Open()` AND keep enforcing them in `Update()`.
- **`Input.ResetInputAxes()` each frame while active** — otherwise WASD held during typing keeps moving the player after the panel closes.
- **`player.SetControllable(false)` on Open / `true` on Close** — stops attack/jump from firing through to the world while the panel has focus.
- **ESC to close** — `if (Input.GetKeyDown(KeyCode.Escape)) Close();` in `Update`. Don't rely on the close button alone; users expect ESC.
- **ESC also opens the game's pause menu on top** — the pause menu is an `NGuiAction` bound to `playerInput.Menu` processed in `GUIWindowManager.Update()`. While your IMGUI panel is open, set `ui.windowManager.IsInputLocked = true` (public field; unset on close). That's the exact lock vanilla's key-rebinding popup (`XUiC_OptionsControlsNewBinding`) uses while capturing a key, and it also silences chat/toolbelt/other global hotkeys. Two gotchas: (1) unlock **one frame after** your panel closes on ESC — if you unlock the same frame, `Menu.WasPressed` is still true and the pause menu can still fire depending on script update order; (2) `SetControllable(false)` does NOT stop the pause menu — it only gates `PlayerMoveController`, not `GUIWindowManager`'s global actions. (Drone-flight-style full HUD hide also blocks it, since the Menu action is gated on `!IsFullHUDDisabled()`.) Learned in FPV's options dialog, 2026-07.
- **Gate your open-hotkey on game menus** — raw `Input.GetKeyDown` fires while the crafting search box has focus or any game menu is open. Guard with `!ui.windowManager.IsModalWindowOpen() && !ui.windowManager.IsInputActive()` before reacting to the key.
- **Eat the event at the end of `OnGUI`** — `if (Event.current is { isKey: true } or { isMouse: true } or …) Event.current.Use();` so IMGUI input doesn't trigger map/inventory shortcuts. (This still doesn't block parallel `Input.*` readers — see above.)
