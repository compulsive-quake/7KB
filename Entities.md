# Entities

Part of the [7DTD Modding Knowledgebase](README.md). Covers custom entity classes (NPCs, animals, zombies) defined via `Config/entityclasses.xml`.

---

## Defining a Custom Entity

Entities extend existing vanilla entity types. All definitions are XPath patches to `entityclasses.xml`:

```xml
<configs>
    <append xpath="/entity_classes">
        <entity_class name="myCustomZombie" extends="zombieMale">
            <property name="Mesh" value="Assets/Prefabs/Entities/ZombieMale.prefab" />
            <property name="SizeExtentsOverride" value="0.5,1,0.5" />
            <property name="SizeExtentsOverrideScale" value="1.2" />
            <property class="AITask">
                <property name="task1" value="BreakBlock" />
                <property name="task2" value="DestroyArea" />
                <property name="task3" value="ApproachAndAttack" />
            </property>
            <drop event="Destroy" name="foodRottingFlesh" count="0,3" prob="0.5" />
            <drop event="Destroy" name="resourceBone" count="0,2" prob="0.5" />
        </entity_class>
    </append>
</configs>
```

---

## Key Properties

| Property | Notes |
|---|---|
| `extends` | Name of the vanilla entity class to inherit from |
| `SizeExtentsOverride` | Collision box size `"x,y,z"` |
| `SizeExtentsOverrideScale` | Scale multiplier applied on top of extents (1.0 = default) |
| `MaxHealth` | Hit points |

Use `extends` to inherit all AI, animations, and properties from a vanilla entity — then only override what you change.

---

## AI Tasks

The `AITask` property class controls what actions the entity takes:

```xml
<property class="AITask">
    <property name="task1" value="BreakBlock" />
    <property name="task2" value="DestroyArea" />
    <property name="task3" value="Territorial" />
    <property name="task4" value="ApproachAndAttack" />
</property>
```

Common task values: `BreakBlock`, `DestroyArea`, `Territorial`, `ApproachAndAttack`, `Wander`, `Investigate`.

---

## Drops

```xml
<drop event="Destroy" name="itemName" count="min,max" prob="0.0-1.0" />
<drop event="Harvest" name="resourceLeather" count="1,3" prob="0.6" />
```

- `event="Destroy"` — drops when entity is killed
- `event="Harvest"` — drops when player harvests the corpse
- `prob` — probability 0.0–1.0

---

## Size Scaling Variants

To create differently-sized variants of the same entity (e.g. progression tiers):

```xml
<entity_class name="myZombieSmall"  extends="myCustomZombie">
    <property name="SizeExtentsOverrideScale" value="0.8" />
</entity_class>
<entity_class name="myZombieLarge"  extends="myCustomZombie">
    <property name="SizeExtentsOverrideScale" value="1.4" />
</entity_class>
```

---

## Visual Customization Properties

Additional properties for controlling entity appearance (used when modifying existing entities):

```xml
<append xpath="/entity_classes/entity_class[@name='zombieMale']">
    <!-- Custom map/compass icon -->
    <property name="entity_icon" value="ui_game_symbol_zicon" />

    <!-- Tint the entity's texture in a custom color (hex RGB, no #) -->
    <!-- param1 = comma-separated Unity material property names to tint -->
    <property name="XTintColor" value="FF00E8"
              param1="_Color,_TintColor,_EmissiveColor" />

    <!-- Replace one of the entity's textures at runtime -->
    <!-- param1 = material path inside a unity3d bundle -->
    <property name="ReplaceTexture" value="2"
              param1="#@modfolder(MyMod):Resources/bundle.unity3d?Materials/MyMaterial.mat" />
</append>
```

---

## Modifying Vanilla Entities

To change a property on an existing entity:

```xml
<configs>
    <set xpath="/entity_classes/entity_class[@name='playerMale']/effect_group/passive_effect[@name='BagSize']/@value">100</set>
</configs>
```
