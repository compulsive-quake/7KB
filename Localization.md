# Localization

Part of the [7DTD Modding Knowledgebase](README.md). Covers adding translated text strings to a mod.

---

## File Location

`Config/Localization.txt` — a CSV file the game merges with vanilla localization.

---

## Format

```
Key,File,Type,UsedInMainMenu,NoTranslate,english,Context,german,spanish,french,italian,japanese,korean,polish,brazilian,russian,turkish,schinese,tchinese
myKey,blocks,Block,,,My English Text,,Mein Text,...
myWindowTitle,windows,XUi,,,My Window Title
myButtonLabel,windows,XUi,,,Click Me
```

| Column | Notes |
|---|---|
| `Key` | Unique identifier — used in XML as `{myKey}` or `text_key="myKey"` |
| `File` | Category: `blocks`, `windows`, `items`, etc. |
| `Type` | `Block`, `Item`, `XUi`, etc. |
| `UsedInMainMenu` | Usually blank |
| `NoTranslate` | Usually blank |
| `english` | The English string |
| `Context` | Optional — notes for translators, not shown in-game |
| `german`, `spanish`, etc. | Optional translated strings; game falls back to `english` if blank |

---

## Using Keys in XML

In window/UI XML, reference a key as a binding:
```xml
<label text="{myWindowTitle}" />
<button text="{myButtonLabel}" />
```

Or via `text_key` attribute:
```xml
<label text_key="myWindowTitle" />
```

In block definitions:
```xml
<!-- Block display name automatically uses block name as key -->
<block name="MyBlock">
    <!-- Localization key = "MyBlock" for display name -->
    <!-- Localization key = "MyBlock_desc" for description, if defined -->
</block>
```

---

## Inline Item Links

In localization string values, wrap an item name in `<<| |>>` to generate a clickable in-game link to the item's info:

```
xuiCraftingTip,windows,XUi,,,Use <<|resourceWood|>> to build this item.
```

The game renders this as a styled hyperlink to the `resourceWood` item page.

---

## Tips

- Keys are case-sensitive
- If a key is missing, the raw key string is displayed (useful for debugging)
- The CSV must use commas; don't use quotes around values unless the value contains a comma
- The `Context` column is for translator notes — it is not shown in-game
