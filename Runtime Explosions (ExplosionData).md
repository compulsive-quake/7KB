# Runtime Explosions (ExplosionData)

How to trigger a code-driven explosion (damage + particle) at runtime, e.g. a
custom projectile/drone detonating on impact.

## Triggering

```csharp
GameManager.Instance.ExplosionServer(
    worldPos,          // Vector3  absolute world pos
    blockPos,          // Vector3i
    Quaternion.identity,
    explosionData,     // ExplosionData
    ownerEntityId,     // int
    0f,                // delay
    false,             // remove block at expl pos
    sourceItemValue);  // ItemValue, optional
```

### 3.0.0 signature change (vs 2.x)
`ExplosionServer` **dropped its leading `int _clrIdx` parameter** in 3.0.0.
Old call started with a color-index int; new call starts with `worldPos`.
Relatedly, `WorldRayHitInfo.hit.clrIdx` was removed. Code compiled against 2.x
throws/needs updating.

## Building ExplosionData — the silent-no-op gotcha (3.0.0)

`ExplosionData(DynamicProperties props, MinEffectController effects = null)`
reads its values from a **nested `Classes["Explosion"]` sub-block with
UN-prefixed keys**. If that class is absent it **returns immediately**, leaving
all-zero field defaults: `EntityRadius = 0`, `BlockRadius = 0`,
`EntityDamage = 0`. The explosion then runs with no damage and no real blast —
`ExplosionServer` succeeds (no exception), so it fails *silently*.

Pre-3.0.0 builds read flat `"Explosion."`-prefixed keys straight off
`props.Values`, so old mod code like `props.Values["Explosion.EntityDamage"]=…`
compiles fine but produces an **inert explosion** after the 3.0.0 update.

### Correct construction
Put the keys under the `"Explosion"` class (un-prefixed):

```csharp
var props = new DynamicProperties();
void Set(string k, object v) =>
    props.SetValue("Explosion", k, Convert.ToString(v, CultureInfo.InvariantCulture));
Set("ParticleIndex", 5);
Set("EntityDamage", 1680f);
Set("RadiusEntities", 16);     // -> EntityRadius
Set("BlockDamage", 0f);
Set("RadiusBlocks", 0f);
Set("BlastPower", 0);          // physics knockback; 0 = none
var data = new ExplosionData(props, null);
if (data.BuffActions == null) data.BuffActions = new List<string>();
```

`DynamicProperties.SetValue(className, propName, value)` auto-creates the nested
class (`GetOrCreateClass`).

> ⚠️ **`SetValue` runs `ValidateKey`, which throws on any `.` in the key.**
> So you can set `EntityDamage`, `BlastPower`, etc., but NOT dotted keys like
> `DamageBonus.water` (those only come from XML parsing via a different path).
> A throw here aborts the whole `ExplosionData` build; if your detonation code
> catches it and falls back to a cosmetic-only particle, you get a *visual
> explosion that deals no damage and no knockback* — a misleading symptom.
> Just omit dotted bonus keys for a code-built blast.

### Field/key reference (3.0.0)
| XML key (under Explosion) | Field | Notes |
|---|---|---|
| `ParticleIndex` | `ParticleIndex` | Index into `WorldStaticData.prefabExplosions`; visual only spawns when `0 < index < length`. |
| `RadiusEntities` | `EntityRadius` (int) | Entities-damaged radius. |
| `EntityDamage` | `EntityDamage` | Defaults to `20 * EntityRadius` if omitted. |
| `RadiusBlocks` | `BlockRadius` | Block-damage radius. |
| `BlockDamage` | `BlockDamage` | Defaults to `BlockRadius²` if omitted. |
| `BlastPower` | `BlastPower` | Physics knockback (`ApplyExplosionForce`); default 100. |
| `DamageBonus.water` etc. | `damageMultiplier` | Parsed via `new DamageMultiplier(props)`. |

`explode()` → `ExplosionClient(...)` spawns the particle independent of
`BlastPower`, so an entities-only blast (BlastPower 0, BlockRadius 0) still shows
its visual.

See also [[Entity Positions]] (absolute world coords), [[Blocks]],
[[Homing Missiles (lock-on + scripted flight)]] (a full scripted-projectile
consumer of this API; note `RocketTurret`'s launch manager still uses the 2.x
call shape and needs this note's 3.0.0 fixes if revived).
