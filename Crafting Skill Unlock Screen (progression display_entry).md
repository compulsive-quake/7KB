# Crafting Skill Unlock Screen (progression `display_entry`)

Part of the [7DTD Modding Knowledgebase](README.md). How the "Unlocks" list on a
crafting-skill page (Skills → e.g. Robotics) is built from `progression.xml`, and
how to add a mod item to it or relabel a row. Verified against V3.0.0 (b259).

---

## Where it comes from

Each `<crafting_skill>` in `progression.xml` (under `/progression/crafting_skills/`)
holds:

- One `<display_entry>` per **row** shown in the skill's Unlocks list.
- One `<effect_group>` whose `RecipeTagUnlocked` passive effects do the **actual
  unlocking** (the display_entry is cosmetic; it does not unlock anything).

Vanilla Robotics (`craftingRobotics`) example:

```xml
<display_entry item="gunBotT3JunkDrone" unlock_level="76,80,85,90,95,100"/>
...
<effect_group>
  <passive_effect name="RecipeTagUnlocked" operation="base_set" level="76,100" value="1" tags="gunBotT3JunkDrone"/>
  <passive_effect name="CraftingTier"      operation="base_add" level="80,85,90,95,100" value="1,2,3,4,5" tags="gunBotT3JunkDrone"/>
</effect_group>
```

`RecipeTagUnlocked tags="X"` matches the recipe whose **name** is `X` (the recipe
`gunBotT3JunkDrone` has tags `learnable,perkTurrets,workbenchCrafting` — note `X`
is NOT in that tag list, so the match is by recipe *name*). A `learnable` recipe
with no `always_unlocked="true"` stays locked until such an effect fires.

---

## Two forms of `<display_entry>` — and the label gotcha

Parser: `ProgressionFromXml` → `ProgressionClass.DisplayData`. Row label/icon come
from `DisplayData.GetName(level)` / `GetIcon(level)`, bound in
`XUiC_SkillCraftingInfoEntry`. Each row shows exactly **one** name and **one** icon
(picked by current quality tier), never a joined list.

1. **Simple:** `item="someItem"` — `GetName` returns `Item.GetLocalizedItemName()`.
   `item=` is a **single** `ItemClass.GetItemClass(...)` lookup — a comma list does
   **not** work here. When `item=` is set it also auto-adds one unlock entry.
2. **Custom:** `icon="a,b" name_key="k1,k2" has_quality="…"` with nested
   `<unlock_entry item="…" unlock_tier="N"/>`. `name_key` values are localized at
   parse time (`Localization.Get`). `unlock_level` = the quality-tier thresholds
   when `has_quality="true"` (one item across 6 tiers), else one level per entry.

**Key trap:** if `item=` is present, `GetName` uses the item's own name and
**ignores `name_key`**. To show a custom row label (e.g. `"Robotic Drone, FPV
Drone"`) you must DROP `item=` and use a `name_key` whose localized string is the
literal label. `GetName`/`GetIcon` clamp the tier index into the array, so a
single-element `name_key`/`icon` with a 6-value `has_quality` `unlock_level` still
shows that one label/icon on every tier while preserving the quality progression.

---

## Recipe: adding a mod item to an existing skill row (worked example)

Goal: FPV mod's `fpvDroneLauncher` shares the Robotic Drone's lvl-76 row and reads
`Robotic Drone, FPV Drone`. Patch (`Config/progression.xml`):

```xml
<remove xpath="/progression/crafting_skills/crafting_skill[@name='craftingRobotics']/display_entry[@item='gunBotT3JunkDrone']"/>
<insertBefore xpath="/progression/crafting_skills/crafting_skill[@name='craftingRobotics']/effect_group[1]">
    <display_entry icon="gunBotT3JunkDrone" name_key="fpvDroneRoboticsUnlockRow" has_quality="true" unlock_level="76,80,85,90,95,100">
        <unlock_entry item="gunBotT3JunkDrone" unlock_tier="1"/>
        <unlock_entry item="fpvDroneLauncher" unlock_tier="1"/>
    </display_entry>
</insertBefore>
<append xpath="/progression/crafting_skills/crafting_skill[@name='craftingRobotics']/effect_group[1]">
    <passive_effect name="RecipeTagUnlocked" operation="base_set" level="76,100" value="1" tags="fpvDroneLauncher"/>
</append>
```

Plus: a `learnable` recipe named `fpvDroneLauncher`, `UnlockedBy="craftingRobotics"`
on the item, and a `Localization.txt` key `fpvDroneRoboticsUnlockRow`.

**Localization comma gotcha:** the label contains a comma, so the CSV value MUST be
quoted: `fpvDroneRoboticsUnlockRow,progression,Attribute,FALSE,FALSE,"Robotic Drone, FPV Drone"`.

---

## Useful vanilla item names (V3.0.0)

`resourceElectricParts` (Electric Parts), `resourceDuctTape` (Duct Tape),
`gunBotRoboticsParts` (Robotics Parts), `resourceScopeLens` (Lens),
`thrownGrenade` (Grenade), `resourceMechanicalParts` (Mechanical Parts).

All of the above (recipes/progression/items) are **restart-bucket** load-baked XML;
`Localization.txt` is xui (hot-reloadable).
