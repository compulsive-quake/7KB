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

> **Mod controller class names must exactly match the `controller=""` attribute** — `controller="PixelPasteWindowController"` requires a class literally named `PixelPasteWindowController`. The game internally tries **`XUiC_PixelPasteWindowController` first** (prepending `XUiC_`), then falls back to the bare name. So the bare name (`PixelPasteWindowController`) is what needs to be in the class definition.
>
> **`Type.GetType()` cannot find mod assembly types** — the game's XUi controller resolution (`XUiFromXml.setController`) calls `Type.GetType(name)` which only searches `Assembly-CSharp` and mscorlib — not mod DLLs. The fix is Harmony postfixes on all three `Type.GetType` overloads (`string`, `string,bool`, `string,bool,bool`) that retry across `AppDomain.CurrentDomain.GetAssemblies()` when the result is null. Use positional injection (`string __0`) rather than a named param to avoid mismatch with Mono's internal parameter names. See `Scripts/XUiControllerPatch.cs`. A transpiler on `XUiFromXml.setController` does NOT work because the `Type.GetType` call is in a nested helper inside `setController`.

> **`OnPress` fires for any mouse button** — check `_mouseButton` if you need to distinguish left/right click.

> **`GetChildrenByType` returns `XUiController[]`** — cast each element individually since the array type is the base class.

> **`GetParentWindow()` can return null** — always null-check before accessing `.Controller`.

> **`OnOpen()` fires children-first** — child controllers' `OnOpen()` runs before the parent window's `OnOpen()`. If a child (e.g., a grid) needs data populated by the parent, do NOT read it in the child's `OnOpen()`. Instead, have the parent explicitly call a method on the child *after* preparing the data in its own `OnOpen()`.
