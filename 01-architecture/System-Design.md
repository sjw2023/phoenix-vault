---
type: system-design
project: Phoenix
updated: 2026-08-03
engine: Unreal Engine 5 (C++)
tags: [project/phoenix, gamedev, systems, data-model]
---

# Phoenix — System Design

Per-system technical notes and data schemas for **Unreal Engine 5**, ordered by the [[Design-Doc|build order]]. Unifying idea: **every system produces modifiers that feed the stat spine** ([[Architecture]]). Schemas are sketches to firm up just before building each system. Follows [[Coding-Conventions]].

## 0. Stats (the spine)
> [!warning] **Designed, not currently built.** Implementation reset 2026-09-14 — reference copy at git tag `pre-reset`. Rebuild at step 3.

`Stats/StatsComponent.*` + `Stats/StatTypes.h`. A `UStatsComponent` (attach to any actor) holds base values + a `TArray<FStatModifier>` and computes `final = (base + Σ FLAT) * (1 + Σ INCREASED) * Π(1 + each MORE)` per stat, lazily (dirty flag + cache), broadcasting `OnStatsChanged`.

```cpp
UENUM(BlueprintType) enum class EModifierType : uint8 { Flat, Increased, More };

USTRUCT(BlueprintType) struct FStatModifier {
    FName Stat;            // "physical_damage"
    EModifierType Type;    // Flat / Increased / More
    float Value;           // Flat = raw; Increased/More = fraction (0.20 == 20%)
    FName Source;          // who granted it — remove by this
};
```

Canonical stat keys (FName) — keep authoritative:

```
life, mana,
strength, dexterity, intelligence,
physical_damage, fire_damage, cold_damage, lightning_damage,
attack_speed, cast_speed, crit_chance, crit_multiplier,
armour, evasion,
movement_speed
```

Every other system's job: `AddModifier(FStatModifier(<key>, <type>, <value>, <sourceId>))`. Nothing writes numbers any other way.

> **GAS note:** Unreal's Gameplay Ability System is the engine-native version of this (Attribute Sets + Gameplay Effects). Deferred for v1 by decision; `UStatsComponent` is the simple stand-in. Keep FLAT/INCREASED/MORE mapping in mind (≈ Additive/Multiplicative GE modifiers) for an eventual port. See [[Architecture]].

## 4. Loot & items (build step 4)
Data-driven: base types + affix pools = data assets; one system rolls them.

**`UItemDefinition : UPrimaryDataAsset`** (assets under `Content/Data/Items/`)
```cpp
FName ItemId;                 // "short_sword"
FText DisplayName;
FGameplayTag Slot;            // Weapon / Helmet / Body / Gloves / Boots / Ring / Amulet
TArray<FStatModifier> Implicits;   // mods every instance grants
TArray<FGameplayTag> AllowedAffixTags;
int32 LevelReq;
TObjectPtr<UStaticMesh> Mesh;  // or icon texture
```

**`FAffixRow : FTableRowBase`** (a `UDataTable` of affix definitions)
```cpp
FName AffixId;
bool bIsPrefix;               // prefix vs suffix
FName Stat;                   // "fire_damage"
EModifierType ModType;
TArray<FAffixTier> Tiers;     // each: MinLevel, Weight, ValueMin, ValueMax
TArray<FGameplayTag> Tags;    // gates which bases it can roll on
```

**Rarity:** Normal (0 affixes), Magic (1–2), Rare (3–6), Unique (later, hand-authored). A rolled **`UItemInstance : UObject`** = pointer to `UItemDefinition` + rolled affixes with concrete values. When equipped, each implicit/affix becomes an `FStatModifier` with `Source = ItemInstance's FName id`.

**Rolling flow (on enemy death):** roll drop chance → rarity → base (weighted by monster/zone level) → N affixes from allowed pools by weight & tier → build `UItemInstance`.

## 5. Inventory & equipment (build step 5)
- **`UInventoryComponent`** — grid model (data only) + a UMG widget reading it. Keep model and view separate.
- **`UEquipmentComponent`** — one slot per `Slot` tag.
- **Equip:** gather item implicit + affix mods → `Stats->AddModifiers(Mods)` with `Source = item id`.
- **Unequip:** `Stats->RemoveModifiersFromSource(item id)`.

That's the whole stat side of equipment — the payoff for building the spine first.

## 6. Skills (build step 6)
Simpler than POE2 gems for v1: a pool of active skills, 2–3 bound to inputs, **data-defined**.

**`USkillDefinition : UPrimaryDataAsset`**
```cpp
FName SkillId;
FText DisplayName;
TArray<FGameplayTag> Tags;    // attack/spell, projectile/melee/area, fire/cold/... → which stats apply
float BaseDamage;
FName DamageStat;             // "fire_damage"
float ManaCost;
float Cooldown;
float Area;                   // 0 = single target
int32 ProjectileCount;        // 0 = melee
```

At use, the skill reads the caster's `UStatsComponent` for the relevant tagged stats (so item "+10% fire damage" improves a fire skill automatically), applies cooldown/mana. Skill *runtime* (an actor/component or ability) is separate from `USkillDefinition` (the data). **This is the most natural place to adopt GAS later** — each skill becomes a `UGameplayAbility`.

## 7. Passive tree (build step 7)
Small graph (~30–60 nodes) for v1.

**`UPassiveNode : UPrimaryDataAsset`**
```cpp
FName NodeId;
FVector2D Position;          // layout in the tree UMG widget
TArray<FName> Neighbors;     // connected node ids
TArray<FStatModifier> Mods;  // granted when allocated
bool bIsNotable;
```

- **Allocation rule:** allocate only if adjacent to an already-owned node (walk the connected graph). One point per level.
- **Apply:** allocate → `Stats->AddModifiers(Node.Mods)` with `Source = NodeId`. Respec → `RemoveModifiersFromSource(NodeId)`.
- Needs: graph data, allocation validation, a UMG UI. Reuses the entire spine.

## Cross-cutting: everything is a modifier source
| System | Source id | Mods added | Mods removed |
|---|---|---|---|
| Equipment | item instance `FName` | on equip | on unequip |
| Passive tree | `NodeId` | on allocate | on respec |
| Buffs/auras (later) | buff `FName` | on apply | on expire |
| Skills | — (read-only) | — | — (skills *read* stats) |

Building any new number-affecting feature = pick its `Source` id, produce `FStatModifier`s, add on gain / remove on loss. That single pattern is why the stat spine was built first.
