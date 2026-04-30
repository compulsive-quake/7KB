# XUi - Controllers (C#)

Part of the [7DTD Modding Knowledgebase](README.md). Covers the C# side of XUi — controller classes, bindings, events, and lifecycle.

---

## Controller Hierarchy

```
XUiController              ← base class for all UI controllers
  └── XUiC_MyWindow        ← your window controller
  └── XUiC_MyGrid          ← grid/list controller
  └── XUiC_MyRow           ← per-row entry controller
```

### Naming Convention

The `controller="Foo"` attribute in XML maps to the C# class `Foo` — the name must match **exactly**. The game does NOT add any prefix when resolving mod controller classes. (Vanilla game controllers internally use an `XUiC_` prefix, but that is irrelevant for mod assemblies.)

---

## Lifecycle Methods

```csharp
public override void Init()
{
    base.Init();
    // Wire up buttons, initialize state.
    // Called once when the controller is created.
    XUiController btn = GetChildById("btnPrint");
    if (btn != null) btn.OnPress += OnPrintPressed;
}

public override void OnOpen()
{
    base.OnOpen();
    // Called every time the window is opened.
    // Refresh data here.
}

public override void OnClose()
{
    base.OnClose();
    // Called when window is closed.
}
```

---

## Data Bindings

XUi uses a string-based binding system. Labels with `text="{myBinding}"` call `GetBindingValue` on their nearest controller ancestor to resolve the value.

```csharp
// NOTE: GetBindingValue is NOT virtual in XUiController.
// You MUST use 'new' (method hiding), NOT 'override'.
// Using 'override' causes a compile error.
public new bool GetBindingValue(ref string _value, string _bindingName)
{
    switch (_bindingName)
    {
        case "selectedFile":
            _value = selectedFile;
            return true;
        case "dimensions":
            _value = dimensions;
            return true;
        default:
            return base.GetBindingValue(ref _value, _bindingName);
    }
}
```

To push updated values to the UI:
```csharp
RefreshBindings(false);  // false = don't force full rebuild
```

---

## Button Events

```csharp
// In Init():
XUiController btnClose = GetChildById("btnClose");
if (btnClose != null)
    btnClose.OnPress += OnClosePressed;

// Handler signature:
private void OnClosePressed(XUiController _sender, int _mouseButton)
{
    xui.playerUI.windowManager.Close("myWindowGroup");
}
```

---

## Finding Child Controllers

```csharp
// By element name attribute
XUiController child = GetChildById("btnPrint");

// By type (returns array)
XUiC_MyRow[] rows = GetChildrenByType<XUiC_MyRow>(null);

// Walk to parent window
XUiWindow win = GetParentWindow();
XUiC_MyWindow ctrl = win?.Controller as XUiC_MyWindow;
```

---

## Opening and Closing Windows

Open from a block's `OnBlockActivated`:
```csharp
localPlayer.PlayerUI.windowManager.Open("myWindowGroup", true);
```

Close from within a controller:
```csharp
xui.playerUI.windowManager.Close("myWindowGroup");
```

The string passed is the `window_group name` from `xui.xml`, **not** the window name.

---

## Window Group Controller

If your `window_group` has a `controller="..."` attribute, that class handles group-level logic. For simple windows this can be a minimal implementation:

```csharp
public class XUiC_MyWindowGroupController : XUiWindowGroup { }
```

---

## Showing Toasts / Tooltips

```csharp
GameManager.ShowTooltip(xui.playerUI.entityPlayer, "Your message here");
```

---

## Grid Controllers

The grid controller exposes a public method for the parent window to call *after* its data is ready (see `OnOpen()` ordering gotcha below):

```csharp
public class XUiC_MyGridController : XUiController
{
    public override void OnOpen()
    {
        base.OnOpen();
        // Do NOT populate here — parent data isn't ready yet.
        // Parent window controller calls RefreshGrid() after its own OnOpen().
    }

    public void RefreshGrid()
    {
        // Get parent window controller for data
        XUiC_MyWindow windowCtrl = GetParentWindow()?.Controller
            as XUiC_MyWindow;
        if (windowCtrl == null) return;

        List<string> items = windowCtrl.GetItems();

        XUiController[] children = GetChildrenByType<XUiC_MyRowController>(null);
        for (int i = 0; i < children.Length; i++)
        {
            var row = children[i] as XUiC_MyRowController;
            if (row == null) continue;

            if (i < items.Count)
            {
                row.SetData(i, items[i]);
                row.ViewComponent.IsVisible = true;
            }
            else
            {
                row.SetData(-1, null);
                row.ViewComponent.IsVisible = false;
            }
        }
    }
}
```

---

## Row (Entry) Controllers

```csharp
public class XUiC_MyRowController : XUiController
{
    private int index = -1;
    private string displayName = "";

    public override void Init()
    {
        base.Init();
        OnPress += OnRowPressed;
    }

    public void SetData(int idx, string name)
    {
        index = idx;
        displayName = name ?? "";
        RefreshBindings(false);
    }

    // 'new' not 'override' — GetBindingValue is not virtual
    public new bool GetBindingValue(ref string _value, string _bindingName)
    {
        if (_bindingName == "rowText")
        {
            _value = displayName;
            return true;
        }
        return base.GetBindingValue(ref _value, _bindingName);
    }

    private void OnRowPressed(XUiController _sender, int _mouseButton)
    {
        if (index < 0) return;
        // Notify parent window
        var win = GetParentWindow()?.Controller as XUiC_MyWindow;
        win?.SelectItem(index);
    }
}
```

---

## Recommended Alternative: IMGUI Controllers

Due to persistent text rendering issues in XUi mod windows (labels inside buttons/rects are invisible), **Unity IMGUI (`MonoBehaviour` + `OnGUI`) is the recommended approach** for mod windows with clickable lists or text-heavy UI. See [XUi - Window System.md](XUi%20-%20Window%20System.md) for the full IMGUI pattern and code template.

Key differences from XUi controllers:
- Extends `MonoBehaviour`, not `XUiController`
- No XML registration — no `windows.xml`, no `xui.xml`
- Open from blocks via `MyBrowserIMGUI.Instance.Open()` instead of `windowManager.Open()`
- Uses singleton pattern with `DontDestroyOnLoad`
- Must lock player controls manually (`SetControllable(false)`) and show cursor
- Background thread callbacks need `MainThreadDispatcher` (must be initialized in `IModApi.InitMod`)

---

## Gotchas & Lessons Learned

> **`GetBindingValue` uses `new`, not `override`** — the base class method is not marked `virtual`. Attempting `override` causes CS0506 compile error.
>
> **`{binding}` text does NOT work for mod controllers** — because `GetBindingValue` is not virtual, the XUi framework calls the base class version (through `XUiController`-typed references), so `new` method hiding never dispatches to the mod's override. **Workaround:** set label text directly via `XUiV_Label.Text`:
> ```csharp
> XUiController lblCtrl = GetChildById("lblMyLabel");
> if (lblCtrl != null)
> {
>     XUiV_Label label = lblCtrl.ViewComponent as XUiV_Label;
>     if (label != null) label.Text = "my value";
> }
> ```
> Use `text=""` (empty) in the XML instead of `text="{binding}"`. Keep the `GetBindingValue` method with `new` as a fallback, but do not rely on it.

> **Controller names in XML must use `"Namespace.ClassName, AssemblyName"` format** — e.g., `controller="SevenAmazon.SevenAmazonShop, 7Amazon"`. If your controllers are in a namespace, you must include it. If they are NOT in a namespace (global), use `controller="ClassName, AssemblyName"`. The assembly name is the DLL name without `.dll`. Without the assembly qualifier, the game only searches `Assembly-CSharp` and won't find mod types.
>
> **Mod controller class names must exactly match the `controller=""` attribute** — `controller="PixelPasteWindowController"` requires a class literally named `PixelPasteWindowController`. The game internally tries **`XUiC_PixelPasteWindowController` first** (prepending `XUiC_`), then falls back to the bare name. So the bare name (`PixelPasteWindowController`) is what needs to be in the class definition.
>
> **`Type.GetType()` cannot find mod assembly types** — the game's XUi controller resolution (`XUiFromXml.setController`) calls `Type.GetType(name)` which only searches `Assembly-CSharp` and mscorlib — not mod DLLs. The fix is to register an `AppDomain.CurrentDomain.AssemblyResolve` handler in `InitMod` so that `Type.GetType("ClassName, YourAssemblyName")` resolves correctly. The game tries `Type.GetType(name + ", Assembly-CSharp")` first, then `Type.GetType(name)` — the AssemblyResolve handler ensures it can find types in your mod DLL. **Important:** controller class names that start with a digit (e.g. `7AmazonShop`) will fail `Type.GetType()` resolution even with the handler. Use alphanumeric names like `SevenAmazonShop` instead. Alternative fix: Harmony postfixes on `Type.GetType` overloads that retry across `AppDomain.CurrentDomain.GetAssemblies()`.
>
> **Prefer native XUi windows over IMGUI** — always build mod UIs as XUi windows rather than Unity IMGUI (`OnGUI`). XUi windows get Escape-to-close, cursor management, input blocking, and controller support for free. IMGUI windows bypass the game's input system entirely, making it nearly impossible to prevent Escape from also opening the pause menu.

> **`OnPress` fires for any mouse button** — check `_mouseButton` if you need to distinguish left/right click.

> **`GetChildrenByType` returns `XUiController[]`** — cast each element individually since the array type is the base class.

> **`GetParentWindow()` can return null** — always null-check before accessing `.Controller`.

> **Setting sprite fill (progress bars)** — `XUiV_Sprite` has a `Fill` property (0.0–1.0) for `type="filled"` sprites. Access it via `GetChildById`:
> ```csharp
> XUiController ctrl = GetChildById("sprFill");
> XUiV_Sprite spr = ctrl?.ViewComponent as XUiV_Sprite;
> if (spr != null) spr.Fill = 0.75f;
> ```

> **Setting sprite color at runtime** — `XUiV_Sprite` color can be changed via the `Color` property. Use reflection if the property name varies by game version (`Color`, `color`, `TintColor`).

> **The `[Preserve]` attribute** — Add `[Preserve]` (from `UnityEngine.Scripting`) to any class referenced only via XML `controller=""` attributes or reflection. Unity's IL stripping may remove classes that aren't directly referenced from code.

> **`OnOpen()` fires children-first** — child controllers' `OnOpen()` runs before the parent window's `OnOpen()`. If a child (e.g., a grid) needs data populated by the parent, do NOT read it in the child's `OnOpen()`. Instead, have the parent explicitly call a method on the child *after* preparing the data in its own `OnOpen()`.

## iOS-style toggle (slide switch) pattern

Use this when you want a binary on/off control that visibly slides like an iOS toggle. Layout: a button hosts two pairs of sprites (track + knob) that the controller swaps via `XUiV_Sprite.IsVisible`. The whole row is the button so the click target is generous.

### XML

Track is `menu_empty3px` (rounded-corner sliced rect). Knob is `ui_game_filled_circle` (truly round). Toggle widget on the LEFT of the label is the conventional layout we use; flip the x-coordinates if you want it on the right.

```xml
<button name="myToggle" pos="20,-104" width="240" height="32" depth="3"
        style="press, hover" hoverscale="1.02" sound="[paging_click]">
    <sprite depth="0" name="rowBg" pos="0,0" width="240" height="32"
            color="30,30,40,200" type="sliced" sprite="menu_empty" />
    <!-- Track: rounded pill, 54×24 starting at x=4 -->
    <sprite depth="1" name="trackOff" pos="4,-4" width="54" height="24"
            color="70,70,85,255"   type="sliced" sprite="menu_empty3px" />
    <sprite depth="1" name="trackOn"  pos="4,-4" width="54" height="24"
            color="80,180,100,255" type="sliced" sprite="menu_empty3px" visible="false" />
    <!-- Knob: 20×20 circle. Off-x=6, On-x=36 (24 px slide inside the track). -->
    <sprite depth="2" name="knobOff" pos="6,-6"  width="20" height="20"
            color="240,240,245,255" sprite="ui_game_filled_circle" />
    <sprite depth="2" name="knobOn"  pos="36,-6" width="20" height="20"
            color="255,255,255,255" sprite="ui_game_filled_circle" visible="false" />
    <label name="myLabel" pos="68,-6" width="160" height="22"
           font_size="15" color="240,240,240,255" justify="left"
           text_key="MyToggleLabel" depth="3" />
</button>
```

### Controller

```csharp
struct Toggle {
    public XUiV_Sprite TrackOff, KnobOff, TrackOn, KnobOn;
    public void Apply(bool on) {
        if (TrackOff != null) TrackOff.IsVisible = !on;
        if (KnobOff  != null) KnobOff.IsVisible  = !on;
        if (TrackOn  != null) TrackOn.IsVisible  = on;
        if (KnobOn   != null) KnobOn.IsVisible   = on;
    }
}

bool _wired;
Toggle _myToggle;

public override void Init() {
    base.Init();
    if (_wired) return;     // see "double-fire" gotcha below
    _wired = true;

    var btn = GetChildById("myToggle");
    if (btn != null) {
        _myToggle.TrackOff = btn.GetChildById("trackOff")?.ViewComponent as XUiV_Sprite;
        _myToggle.TrackOn  = btn.GetChildById("trackOn")?.ViewComponent  as XUiV_Sprite;
        _myToggle.KnobOff  = btn.GetChildById("knobOff")?.ViewComponent  as XUiV_Sprite;
        _myToggle.KnobOn   = btn.GetChildById("knobOn")?.ViewComponent   as XUiV_Sprite;
        btn.OnPress += (_, __) => {
            MyState = !MyState;
            _myToggle.Apply(MyState);
        };
    }
}

public override void OnOpen() {
    base.OnOpen();
    _myToggle.Apply(MyState);   // sync visuals from the source of truth
}
```

### Gotchas

- **Double-fire on click**: `Init()` can run more than once for the same controller (XUi may re-parse the window or re-call Init on first OnOpen). Without a `_wired` flag every press fires twice (fields toggle, then toggle back, net no change). Symptom in logs: two consecutive `[mod] toggle Foo ok=True new=True` / `new=False` pairs ~2 ms apart.
- **OnPress signature** is `(XUiController _sender, int _mouseButton)` — the int is which mouse button (0/1/2), *not* a press-state boolean. A single click fires it once.
- **Pivot**: leaving `pivot` unset works for these full-row buttons; do not set `pivot="topleft"` unless your child positions match that origin.
- **Hover scaling**: `hoverscale="1.0"` may make NGUI skip the hover state and break input. Use `hoverscale="1.02"` (almost imperceptible) when you want minimal hover animation but reliable click handling.
- **Track length vs knob slide**: knob On-x = trackPos.x + (trackWidth − knobWidth − inset). For a 54-wide track with 20-wide knob and 6 px inset on each side: knob-off-x=trackX+2, knob-on-x=trackX+32. Adjust if you change track width.
- **Source of truth**: keep the toggle state in a static field / model object. The controller only renders. After every press, call `Apply()` so the visuals match the new value.

## ⚠️ Don't declare `controller=` on both the window_group AND the window

A window_group registered in `xui.xml` and the matching `<window>` definition in `windows.xml` will both instantiate their own controller if both carry a `controller="..."` attribute. The button OnPress events end up wired *twice* — once per instance — and every click flips the state twice for a net no-op. Symptom in logs: `OnOpen` (or any Init log) firing twice per app open, and every press logging twice ~1 ms apart.

**Single source of truth**: declare the controller on the window_group only, leave it off the `<window>` element.

```xml
<!-- xui.xml -->
<window_group name="zPhoneBandits" controller="BanditsAppController, zPhone">
    <window name="zPhoneBandits" />
</window_group>

<!-- windows.xml — NO controller= here -->
<window name="zPhoneBandits" width="280" height="900" ...>
    ...
</window>
```

This applies to any window — not just toggles. Buttons that just open another window won't visibly misbehave from the duplicate (calling `Open` twice is idempotent), so the bug hides until you have a toggle or counter that flips.
