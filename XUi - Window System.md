# XUi - Window System

Part of the [7DTD Modding Knowledgebase](README.md). Covers the XML side of 7DTD's XUi UI framework — element types, attributes, layout, and rendering.

---

## How Windows Are Registered

Windows are added in two places:

**1. `Config/XUi/windows.xml`** — defines the window element tree (layout, visuals, controllers)

```xml
<configs>
    <append xpath="/windows">
        <window name="myWindow" width="700" height="500"
                pos="0,0" anchor="CenterCenter" panel="Center"
                controller="MyWindowController"
                cursor_area="true">
            <!-- elements here -->
        </window>
    </append>
</configs>
```

**2. `Config/XUi/xui.xml`** — registers the window in the XUi ruleset so it can be opened

```xml
<configs>
    <append xpath="/xui/ruleset[@name='default']">
        <window_group name="myWindowGroup" controller="MyWindowGroupController">
            <window name="myWindow" />
        </window_group>
    </append>
</configs>
```

> The `window_group` name is what you pass to `windowManager.Open("myWindowGroup")`.

---

## Window Attributes

| Attribute | Values | Notes |
|---|---|---|
| `name` | string | Unique identifier |
| `width` / `height` | int | Pixel dimensions |
| `pos` | `"x,y"` | Offset from anchor point |
| `anchor` | `CenterCenter`, `LeftTop`, `RightBottom`, etc. | NGUI-style anchor — `CenterCenter` centers on screen |
| `panel` | `"Center"`, `"Left"`, `"Right"` | Which UI panel this window belongs to |
| `controller` | class name (without `XUiC_` prefix) | C# controller class |
| `cursor_area` | `"true"` | Shows cursor when window is open |
| `depth` | int | Z-order |

> `open_anywhere="true"` is **not** a valid attribute — omit it.

---

## Element Types

### `<rect>` — Layout Container

`<rect>` is a **purely structural** container. It has no visual rendering of its own. Do not put `color=` or `sprite=` attributes on it expecting it to render — those are ignored or cause white-box artifacts.

```xml
<rect depth="2" pos="15,-50" width="460" height="400">
    <!-- visual children go here -->
</rect>
```

Use it to group and position child elements.

---

### `<sprite>` — Visual Background / Image

`<sprite>` renders a visual. This is what you use for **colored backgrounds**, borders, icons, and filled bars.

```xml
<!-- Solid dark background filling parent rect -->
<sprite depth="1" color="32,32,32,240" type="sliced" />

<!-- Border-only box using a 9-slice border sprite -->
<sprite depth="1" sprite="menu_empty3px" color="[black]" type="sliced" fillcenter="false" />

<!-- Named-color solid fill -->
<sprite depth="2" color="[darkGrey]" type="sliced" />

<!-- Interactive hover highlight (used inside clickable rows) -->
<sprite depth="1" color="[darkGrey]" type="sliced" on_hover="true" />
```

**Key `type` values:**
- `"sliced"` — 9-slice sprite, scales cleanly to any size (most common for backgrounds)
- `"filled"` — used for progress bars (combine with `fill="0.75"`)

**Key attributes:**

| Attribute | Notes |
|---|---|
| `color` | `"R,G,B,A"` (0–255) or named token like `[black]` |
| `type` | `"sliced"`, `"filled"`, or `"FlippedHorizontal"` |
| `sprite` | Optional named sprite from the UI atlas (e.g. `"menu_empty3px"`) |
| `atlas` | Override sprite atlas: `"UIAtlas"`, `"ItemIconAtlas"`, `"UIHelpWindow"`, `"PrefabImages"`, `"UIMods"` |
| `fillcenter` | `"false"` renders only the border, not the fill (border-box effect) |
| `on_hover` | `"true"` — highlights on mouse-over |
| `on_press` | `"true"` — visual feedback on click |
| `globalopacity` | `"false"` — does NOT inherit parent opacity |
| `globalopacitymod` | Float multiplier applied to inherited opacity |
| `foregroundlayer` | `"true"` — always renders in front of other elements |
| `pivot` | `"center"`, `"left"`, `"right"`, `"top"`, `"bottom"` — rotation/anchor pivot point |
| `rotation` | Degrees (e.g. `"45"`) — rotates around pivot |
| `flip` | `"true"` — mirrors the sprite |

---

### `<filledsprite>` — Progress Bars

```xml
<filledsprite depth="3" color="255,0,0,160" type="filled" fill="{healthPercent}" />
```

`fill` accepts 0–1 float or a binding.

---

### `<label>` — Text

```xml
<label depth="2" pos="10,-12" width="400" height="30"
       text="Hello World" font_size="20"
       color="[white]" />

<!-- Localized text via key -->
<label text="{my_localization_key}" font_size="20" color="[white]" />
```

| Attribute | Notes |
|---|---|
| `text` | Literal string, localization key binding `{key}`, or data binding `{bindingName}` |
| `text_key` | Direct localization key (alternative to `text="{key}"`) |
| `font_size` | Integer |
| `color` | `"R,G,B,A"` or named token |
| `justify` | `"left"`, `"center"`, `"right"` |
| `style` | e.g. `"outline"` adds text shadow |
| `upper_case` | `"true"` forces uppercase |
| `effect` | `"outline"` — outline/shadow effect on text |
| `effect_color` | Color of the effect: `"R,G,B,A"` |
| `effect_distance` | Offset of the effect: `"x,y"` (e.g. `"1,1"`) |
| `overflow` | How to handle text that exceeds bounds |
| `parse_actions` | `"true"` — enables parsing of inline action links |

**Inline text color formatting:**
```xml
<!-- Wrap text in [HEXCODE]...[-] to colorize inline -->
<label text="[DECEA3]Price[-]: 100" />
<label text="[FFD200]Gold[-] or [FF2400]Red[-]" />
```

---

### `<button>` — Clickable Button

```xml
<!-- Icon button -->
<button name="btnCraft" depth="2" pos="30,-20" style="icon30px, press, hover"
        sprite="ui_game_symbol_hammer" pivot="center" sound="[paging_click]" />

<!-- Text button -->
<button name="btnPrint" depth="3" pos="490,-380" width="195" height="50"
        font_size="22" text="Print" sound="craft_click_craft" />
```

- Wired to C# via `GetChildById("btnPrint").OnPress += handler`
- The `style="press"` attribute gives visual press feedback
- Sound plays on click

---

### `<grid>` — Repeating List / Grid

```xml
<grid name="imageGrid" depth="3" pos="5,-30" rows="12" cols="1"
      cell_width="450" cell_height="30"
      controller="PixelPasteImageGridController">
    <!-- template element (one per row) -->
    <rect name="imageEntry" width="450" height="28" depth="4"
          style="press"
          controller="PixelPasteImageEntryController">
        <sprite depth="1" color="[darkGrey]" type="sliced" on_hover="true" />
        <label name="lblText" depth="2" pos="10,-7" width="430" height="24"
               font_size="16" color="[white]" text="{rowBinding}" />
    </rect>
</grid>
```

- The child element is a **template** — cloned once per row
- `rows` × `cols` = max visible entries
- Child controller handles per-row data via `SetFileInfo()` / `RefreshBindings()`
- `repeat_content="true"` — the template row is auto-cloned to fill the grid (alternative to defining each row manually)
- `arrangement="vertical"` (default) fills top-to-bottom; `arrangement="horizontal"` fills left-to-right
- Pair with a `<textfield search_field="true">` sibling to get live text filtering for free

---

### `<panel>` — Themed Container

A higher-level container that supports the game's theming system.

```xml
<panel name="header" height="43" depth="1" backgroundspritename="ui_game_panel_header">
    <label style="header.name" text="{title}" />
</panel>
```

Used for headers and standard game UI chrome.

---

### `<texture>` — Raw Texture Rendering

Renders a Unity texture directly, useful for custom images loaded from mod resources.

```xml
<texture name="preview" pos="0,-10" width="128" height="128"
         material="Materials/Transparent Colored"
         depth="3" />
```

| Attribute | Notes |
|---|---|
| `texture` | Texture asset path or binding |
| `material` | Unity material (e.g. `"Materials/Transparent Colored"`) |
| `original_aspect_ratio` | `"true"` preserves aspect ratio on resize |
| `color` | Tint color |
| `globalopacity` | Whether to inherit global UI opacity |

---

### `<textfield>` — Text Input

```xml
<textfield name="searchBox" pos="5,-5" width="280" height="36"
           font_size="18" depth="3"
           search_field="true"
           focus_on_open="true"
           character_limit="64" />
```

| Attribute | Notes |
|---|---|
| `search_field` | `"true"` — connects to sibling grid for live filtering |
| `focus_on_open` | `"true"` — auto-focuses when window opens |
| `character_limit` | Max character count |
| `virtual_keyboard_prompt` | Prompt text for controller virtual keyboard |
| `open_vk_on_open` | `"true"` — opens virtual keyboard on open |
| `close_group_on_tab` | `"true"` — Tab closes the window group |
| `clear_button` | `"true"` — shows an X button to clear text |

---

### `<combobox>` — Numeric Stepper / Dropdown

```xml
<combobox name="qualityPicker" pos="10,-10" width="120" height="36"
          type="ComboBoxInt"
          format_string="{0}"
          value_min="1" value_max="6"
          value_wrap="true" value_increment="1" />
```

---

### `<togglebutton>` — Toggle Button

```xml
<togglebutton name="toggleLocked" pos="10,-10" width="160" height="36"
              caption_key="btnLockLabel" tooltip_key="btnLockTooltip"
              depth="3" style="default" />
```

---

### `<pager>` / `<scroll_paging>` — Pagination Controls

```xml
<!-- Basic pager linking to a grid -->
<pager name="listPager" pos="0,-5" contents_parent="imageGrid" />

<!-- Pager with page number display -->
<scroll_paging pos="0,-420" width="460" height="36"
               show_max_page="true" separator=" / "
               primary_pager="listPager"
               contents_parent="imageGrid"
               visible_max_rows="12" />
```

- `contents_parent` — the `name` of the `<grid>` this pager controls
- `visible_max_rows` — how many rows are shown per page

---

## Named Color Tokens

These are defined globally in `styles.xml` and usable anywhere with `color="[tokenName]"`:

| Token | Approximate color |
|---|---|
| `[white]` | White |
| `[black]` | Black |
| `[lightGrey]` | Light grey |
| `[mediumGrey]` | Medium grey |
| `[darkGrey]` | Dark grey |
| `[red]` | Red |
| `[beige]` | Warm off-white |
| `[iconColor]` | Default icon tint |
| `[labelColor]` | Default label text |
| `[disabledLabelColor]` | Greyed-out label |

---

## Layout Rules

- `pos="x,y"` — offset from the **top-left** of the parent element
- **Y is negative going down**: `pos="10,-50"` means 10px right, 50px down
- `depth` controls draw order (higher = drawn on top)
- `anchor` on a `<window>` controls which point of the screen it's pinned to

---

## Binding Expressions

Standard bindings use `{bindingName}` to resolve a string from the nearest controller. For complex logic, use the expression syntax `{# ... }`:

```xml
<!-- Simple binding -->
<label text="{itemcount}" />

<!-- Negation -->
<sprite visible="{# !showEmpty}" />

<!-- String comparison -->
<label visible="{# prefabNight == ''}" />

<!-- Numeric comparison with cast -->
<button visible="{# int(pageNumber) > 1}" />

<!-- Ternary operator -->
<label justify="{# hasDurability ? 'Center' : 'Right'}" />
```

Expression syntax supports: `!`, `==`, `!=`, `>`, `<`, `>=`, `<=`, `&&`, `||`, ternary `? :`, and the `int()` cast function.

---

## Custom Controls (Template Variables)

Controls defined in `controls.xml` support `${var}` template variables for parameterized instances:

```xml
<!-- Define in controls.xml -->
<controls>
    <my_stat_row depth="${depth}" pos="${pos}" width="${width}" height="${height}"
                 visible="${visible}">
        <rect pos="${pos}" width="${width}" height="${height}">
            <label text="${label}" font_size="${font_size}" />
        </rect>
    </my_stat_row>
</controls>

<!-- Use in windows.xml — pass variable values as attributes -->
<my_stat_row name="row0" depth="3" pos="0,-50" width="400" height="30"
             label="Damage" font_size="18" visible="true" />
```

Template variables only work as attribute values — they cannot be used in XPath expressions.

---

## Conditional Patching

Apply patches only when certain mods are (or aren't) loaded:

```xml
<configs>
    <conditional>
        <if cond="mod_loaded('OtherModName')">
            <!-- These patches only apply if OtherModName is loaded -->
            <set xpath="/xui/ruleset[@name='default']/@scale">1.245</set>
        </if>
    </conditional>

    <conditional>
        <if cond="not mod_loaded('OtherModName')">
            <!-- These patches only apply if OtherModName is NOT loaded -->
            <append xpath="/windows">
                <window name="myFallbackWindow" ... />
            </append>
        </if>
    </conditional>
</configs>
```

This is how UI overhaul mods handle compatibility with each other.

---

## Common Patterns

### Dark background panel with border
```xml
<sprite depth="1" sprite="menu_empty3px" color="[black]" type="sliced" fillcenter="false" />
<sprite depth="2" color="32,32,32,240" type="sliced" />
```

### Clickable list row with hover highlight
```xml
<rect name="rowEntry" width="450" height="28" depth="4" style="press"
      controller="MyRowController">
    <sprite depth="1" color="[darkGrey]" type="sliced" on_hover="true" />
    <label name="lblText" depth="2" pos="10,-7" width="430" height="24"
           font_size="16" color="[white]" text="{rowText}" />
</rect>
```

### Section header bar
```xml
<rect depth="2" width="700" height="40" pos="0,0">
    <sprite depth="1" color="20,20,20,255" type="sliced" />
    <label depth="2" pos="15,-12" width="500" height="30"
           text="Section Title" font_size="24" color="[white]" />
</rect>
```

---

## Controls (Reusable Templates)

Reusable UI components are defined in `Config/XUi/controls.xml` (or `XUi_Common/controls.xml` for cross-context sharing). These work like named templates you can reference from windows.

```xml
<!-- Define a reusable component in controls.xml -->
<controls>
    <include name="my_row_entry">
        <rect width="450" height="28">
            <sprite depth="1" color="[darkGrey]" type="sliced" on_hover="true" />
            <label name="lblText" depth="2" pos="10,-7" width="430" height="24"
                   font_size="16" color="[white]" text="{rowText}" />
        </rect>
    </include>
</controls>
```

Then reference it in `windows.xml` with the element name matching the `name` attribute:
```xml
<my_row_entry name="row0" />
```

---

## Custom Style Tokens (styles.xml)

You can define custom named color tokens in `Config/XUi/styles.xml` (or `XUi_Common/styles.xml`) and then use them like built-in tokens:

```xml
<styles>
    <style name="MyDarkRed" color="42,16,16,255" />
    <style name="MyWindowBg" color="0,0,0,220" />
</styles>
```

Use in windows:
```xml
<sprite color="[MyDarkRed]" type="sliced" />
```

---

## Removing Existing Windows

To remove a vanilla window (e.g. to replace it with a custom version), use a `<remove>` XPath patch in your `windows.xml`:

```xml
<configs>
    <remove xpath="/windows/window[@name='windowToolbelt']" />
    <append xpath="/windows">
        <window name="windowToolbelt" ...>
            <!-- your replacement -->
        </window>
    </append>
</configs>
```

This is how Advanced UI mods fully replace HUD elements.

---

## XUi Ruleset Scale

In `xui.xml` you can set the global UI scale for the ruleset:

```xml
<configs>
    <set xpath="/xui/ruleset[@name='default']/@scale">1.14</set>
</configs>
```

---

## Gotchas & Lessons Learned

> **Grid templates are NOT auto-cloned without `repeat_content="true"`** — without this attribute the template child element exists exactly once, so `GetChildrenByType<RowController>` finds only 1 entry regardless of `rows=`. Always add `repeat_content="true"` to `<grid>` elements that use a single template row. Also set `visible="false"` on the template so empty rows stay hidden until `SetData` makes them visible.

> **`<rect>` does not render visually.** Putting `color=` or `sprite=` on a `<rect>` does nothing useful. Use a `<sprite>` child element inside the rect for any background fill.

> **Adding `sprite="UISprite"` to a `<rect>` will render a plain white box** that covers everything underneath, since the `color` tint is not applied to rect elements — only to `<sprite>` elements.

> **`open_anywhere="true"` is not a valid window attribute** in 7DTD XUi. Remove it.

> **`GetBindingValue` in `XUiController` is not virtual.** You must use `new` (method hiding), not `override`. The game resolves it via the concrete type. Using `override` causes a compile error.

> **Controller class naming**: The `controller="Foo"` attribute in XML maps to the C# class `Foo` exactly — no prefix is added for mod assemblies. Your class must be named `Foo`, not `XUiC_Foo`.

> **`color` on `<button>` does NOT tint the background.** Buttons always render with their default white/light sprite regardless of the `color` attribute. To get readable text on buttons, use a separate `<label>` child with `color="[black]"` (dark text on light button) rather than relying on the button's own text rendering or background tinting. For colored swatches (e.g. a palette), use `<rect style="press">` with a `<sprite color="R,G,B,A" type="sliced" />` child instead of a `<button>` — sprites properly render the color, and `style="press"` makes the rect clickable via `OnPress`.
