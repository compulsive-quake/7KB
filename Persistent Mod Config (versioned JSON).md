# Persistent Mod Config (versioned JSON)

How to persist user settings that survive a mod redeploy, and evolve the schema
over time without wiping what the player already tuned. Reference implementation:
`FPV/src/FPVControlsConfig.cs`.

## Store outside the mod folder

The `Mods/<mod>/` folder is **overwritten on every deploy**, so anything written
there is lost. Persist under the user data dir instead:

```csharp
Path.Combine(
    Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), // %AppData%
    "7DaysToDie", "FPVMod", "controls.json");
```

Serialize with `Newtonsoft.Json` (`JsonConvert.SerializeObject(this, Formatting.Indented)`).
Wrap load/save in try/catch and fall back to `Default()` — a corrupt/partial file
must never hard-fail the mod.

## Load pipeline

`LoadOrDefault()` runs, in order:

1. `FillMissing()` — backfill any null field/list from `Default()`, clamp numeric
   ranges, resize fixed arrays (`FitBoolArray`). A hand-edited or older JSON can
   leave members null; the runtime must never deref a missing one.
2. One `UpgradeXxx()` per schema axis (see below).

Each `Default()` stamps every `CurrentXxxVersion` so a **fresh** config skips all
migrations.

## Versioned migrations (the key pattern)

Each independent schema concern gets its own version pair:

```csharp
private const int CurrentProfilesVersion = 1;   // code baseline
public  int ProfilesVersion = 0;                // serialized; 0 = pre-versioning
```

An `UpgradeXxx()` early-returns when `XxxVersion >= CurrentXxxVersion`, otherwise
mutates in place, sets the version, and `Save()`s so it runs **exactly once**:

```csharp
private void UpgradeProfiles() {
    if (ProfilesVersion >= CurrentProfilesVersion) return;
    // ...migrate...
    ProfilesVersion = CurrentProfilesVersion;
    Save();
}
```

FPV already carries four: `TuningVersion` (non-user physics fields overwritten to
the new baseline), `AxisSchemaVersion` (winmm→HID axis-index remap clears stale
binds), `KeySchemaVersion` (fix an inverted default bind only if unchanged), and
`ProfilesVersion` (below).

### Renaming/splitting a field without data loss

To move an old field's value somewhere new: keep the **old field as a nullable
"legacy" member** (no initializer), read it once in the upgrade, then null it so
it neither re-migrates nor clutters future JSON. Example — splitting one shared
rateprofile+HUD into per-drone `Bomb`/`Tiny` profiles:

```csharp
public FPVDroneProfile Bomb = new FPVDroneProfile();
public FPVDroneProfile Tiny = new FPVDroneProfile();
public FPVRateProfile   Rates;   // legacy → migrate to Bomb/Tiny, then null
public FPVHudSettings    Hud;     // legacy → same

private void UpgradeProfiles() {
    if (ProfilesVersion >= CurrentProfilesVersion) return;
    if (Rates != null) { Rates.FillMissing(); Bomb.Rates = Rates; Tiny.Rates = Rates.Clone(); Rates = null; }
    if (Hud   != null) { Bomb.Hud = Hud; Tiny.Hud = Hud.Clone(); Hud = null; }
    ProfilesVersion = CurrentProfilesVersion; Save();
}
```

Gotchas:
- Missing JSON keys keep the C# field initializer, so `Bomb`/`Tiny` deserialize as
  fresh defaults on an old file; the upgrade then overwrites them from the legacy
  value. `FillMissing()` runs **before** the upgrade, so filled defaults are safe.
- **Clone** when seeding two destinations from one legacy object, or they alias and
  edits to one bleed into the other. Give each settings class a deep `Clone()`.
- Legacy nulls still serialize (`"Rates": null`) — harmless; ignore or add
  `NullValueHandling.Ignore` if you care about tidiness.

## Per-drone / per-variant profiles

When one config drives several item variants that share a transmitter but want
their own feel: keep the physical-input stuff shared (bindings, deadzone, gamepad
calibration, flight mode) and bundle the per-variant stuff into a small profile
class selected by an enum key:

```csharp
public enum FPVDroneKind { Bomb = 0, Tiny = 1 }
public FPVDroneProfile GetProfile(FPVDroneKind kind) => kind == FPVDroneKind.Tiny ? Tiny : Bomb;
public static FPVDroneKind KindForItem(string itemName) =>
    itemName == "fpvTinyDrone" ? FPVDroneKind.Tiny : FPVDroneKind.Bomb;   // default = safe base
```

Derive the kind from the **item identity** (`ItemClass.Name`, available as
`data.invData.item.Name` in an `ItemAction` and `inventory.holdingItem.Name` on
foot), thread it through the launch call into the runtime, and read
`GetProfile(kind).Xxx` at every flight/HUD site. The shared Options dialog edits
whichever variant a small selector points at (defaulting to the held item's kind).

See also: [Betaflight Rates Math](Betaflight%20Rates%20Math.md),
[Items](Items.md).
