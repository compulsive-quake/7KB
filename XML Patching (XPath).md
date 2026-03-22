# XML Patching (XPath)

Part of the [7DTD Modding Knowledgebase](README.md). Covers the XPath patch system used by all `Config/` XML files.

---

## Overview

7DTD mods don't replace vanilla XML files — they **patch** them. Each `Config/*.xml` file is an XPath patch document. The game merges patches from all mods at startup, then discards the patch files and uses the merged result.

---

## Basic Patch Operations

### Append a child node

```xml
<configs>
    <append xpath="/blocks">
        <block name="myBlock">
            <property name="Class" value="MyBlock" />
        </block>
    </append>
</configs>
```

### Set an attribute on an existing node

```xml
<configs>
    <set xpath="/blocks/block[@name='stone']/property[@name='MaxDamage']/@value">5000</set>
</configs>
```

### Insert before a node

```xml
<configs>
    <insertBefore xpath="/blocks/block[@name='stone']">
        <block name="myBlock">...</block>
    </insertBefore>
</configs>
```

### Remove a node

```xml
<configs>
    <remove xpath="/blocks/block[@name='stone']" />
</configs>
```

Works for any node type — confirmed working on vanilla window definitions and item/entity entries.

### Set an attribute (alternative form)

`<setattribute>` modifies a named attribute on the matched element:

```xml
<configs>
    <setattribute xpath="/blocks/block[@name='stone']" name="MaxDamage">5000</setattribute>
</configs>
```

Equivalent to `<set xpath=".../@attrName">value</set>`, but more explicit.

### Insert after a node

```xml
<configs>
    <insertAfter xpath="/blocks/block[@name='stone']">
        <block name="myBlock">...</block>
    </insertAfter>
</configs>
```

---

## Targeting Vanilla XUi

To add a window:
```xml
<configs>
    <append xpath="/windows">
        <window name="myWindow" ...>
            ...
        </window>
    </append>
</configs>
```

To add a window group to the default ruleset:
```xml
<configs>
    <append xpath="/xui/ruleset[@name='default']">
        <window_group name="myGroup" controller="MyGroupController">
            <window name="myWindow" />
        </window_group>
    </append>
</configs>
```

---

## Targeting Blocks

```xml
<configs>
    <append xpath="/blocks">
        <block name="PixelPaste">
            <property name="Class" value="PixelPaste" />
            <property name="Material" value="Mconcrete" />
            <!-- ... -->
        </block>
    </append>
</configs>
```

---

## Conditional Patching

Apply patches only when another mod is (or isn't) loaded:

```xml
<configs>
    <conditional>
        <if cond="mod_loaded('XRamosZUI')">
            <!-- Only applied if XRamosZUI is installed -->
            <set xpath="/xui/ruleset[@name='default']/@scale">1.245</set>
        </if>
    </conditional>

    <conditional>
        <if cond="not mod_loaded('OtherMod')">
            <!-- Only applied if OtherMod is NOT installed -->
        </if>
    </conditional>
</configs>
```

The mod name in `mod_loaded()` is the `Name` value from that mod's `ModInfo.xml`.

---

## Advanced XPath Selectors

```xml
<!-- Double-slash: recursive descent — match anywhere in tree -->
<remove xpath="//window[@name='windowFoo']" />

<!-- contains() — partial attribute match -->
<set xpath="//item_stack[contains(@controller, 'ItemStack')]/@tooltip">""</set>

<!-- Multiple attribute predicates -->
<set xpath="/items/item[@name='ammo9mmBullet']/property[@name='Stacknumber']/@value">500</set>
```

---

## Tips

- The `xpath` attribute is standard XPath 1.0
- Attribute selectors use `[@name='value']` syntax
- Multiple operations can be in one file — list them all inside `<configs>`
- Patches are applied in load order; mod load order is not guaranteed unless using `ModInfo.xml` dependencies
- The same `<remove>` + `<append>` pattern works for replacing vanilla XUi windows, items, entity classes, etc.

---

## Sounds

Custom sounds go in `Config/sounds.xml`:

```xml
<configs>
    <append xpath="/SoundDataNode">
        <SoundDataNode name="MySoundName">
            <Noise ID="6" noise="250" time="3" heat_map_strength="5" />
            <AudioClip ClipName="#@modfolder:Resources/MySounds.unity3d?MySoundClip.wav" Loop="false" />
        </SoundDataNode>
    </append>
</configs>
```

- `#@modfolder:` resolves to the mod's install directory
- Audio must be inside a `.unity3d` asset bundle in the `Resources/` folder
- `heat_map_strength` controls how much noise the sound generates (attracts zombies)
- `Loop="true"` for ambient/looping sounds
