---
type: design-doc
project: Phoenix
version: 0.3 draft
updated: 2026-09-14
engine: Unreal Engine 5.8 (C++)
view: 3D top-down
platform: macOS
tags: [project/phoenix, gamedev, gdd]
---

# Phoenix — Design Doc

*A POE2-like action RPG. First game project. Learning-focused.* — Version 0.3 draft · 2026-09-14 · Engine: Unreal Engine 5.8 (C++), 3D top-down · Platform: macOS

## Revision history

| ver | date | what changed |
|---|---|---|
| 0.1 | 2026-07-13 | First draft (Godot era) |
| 0.2 | 2026-08-03 | Engine switched to Unreal Engine 5 |
| **0.3** | **2026-09-14** | v1 renamed from "vertical slice" to **Alpha**; build order superseded by milestones in [[Production-Plan-2026-09-14]]; pillars given tests; platform macOS; UE 5.8; stale Godot and install text struck |

> Superseded text is struck rather than deleted.

## 1. One-line pitch
Phoenix is a top-down action RPG in the spirit of *Path of Exile 2*: you click to move and fight through dark, monster-filled zones, kill packs of enemies, and chase an endless stream of randomized loot that feeds deep character customization through skills and a large passive tree.

## 2. Vision & pillars
The goal is not to clone POE2 — that was built by a large studio over many years — but to build a *real, playable slice* of the ARPG experience that captures why the genre is fun, while being an achievable first project that teaches game architecture, real-time systems, and data-driven design.

Three pillars guide every decision:

- **Satisfying combat.** Moment-to-moment play must feel good on its own — responsive movement, clear hit feedback, enemies that die in satisfying bursts.
- **Meaningful loot.** Items are the heartbeat of an ARPG. Drops must be frequent, varied, and occasionally exciting.
- **Build expression.** Even in a small slice, an active skill plus a handful of passive choices should let two players build noticeably different characters.

Anything that doesn't serve one of these three is out of scope for v1.

**How each pillar is tested** — and at which milestone — is defined in [[Production-Plan-2026-09-14#4. Pillars made testable (fixes G3)|Production Plan §4]].

## 3. Scope (v1 = Alpha: every v1 feature functional)

> [!note] Renamed in v0.3. ~~v1 = vertical slice~~. This list is every v1 feature, which is the industry meaning of **Alpha**. A much smaller **vertical slice** milestone now comes first — see [[Production-Plan-2026-09-14#M2 — Vertical Slice|Production Plan M2]].
**In scope:** a single character in top-down view, click-to-move, one or two active skills, a single zone populated with several enemy kinds spawning in packs, enemies that drop items on death, items with rarity tiers and randomized affixes, an inventory + equipment slots that change stats, a small passive tree (a few dozen nodes) granting stat bonuses per level, and an end boss as a goal.

**Out of scope (later versions):** multiplayer, trading, online features; the full 1000+ node tree; socketed skill gems (v1 uses a simpler skill system); crafting/item modification; multiple acts / campaign story; endgame maps/Atlas; leagues, seasons, economy; voice acting, cutscenes, heavy narrative.

Keeping this line bright is the single most important thing for actually finishing.

## 4. Core gameplay loop
Second-to-second: **move → engage a pack → use skills to kill → collect loot → repeat.** Session-level: **clear zone → level up → find upgrades → equip → allocate passive points → tackle harder content.** For v1, "harder content" = the boss plus an optional difficulty toggle raising monster level and reward quality.

## 5. Systems

### 5.1 Character & stats
Core attributes (POE-flavored): **Strength, Dexterity, Intelligence.** Derived: **Life, Mana, Armour, Attack/Cast Speed, Critical Strike Chance,** and **Damage** split into Physical/Fire/Cold/Lightning, plus movement speed. Key idea: stats are computed by **summing modifiers from all sources** (base + items + passives + buffs) whenever something changes — the stat-aggregation system is the backbone everything plugs into.

### 5.2 Combat
> [!tip] **Full spec:** [[Combat]] (game design) · [[Combat-Tech]] (technical). The paragraph below is the summary.

Real-time, click-driven. Left-click moves/attacks; number/letter keys trigger skills. Damage from attacker stats + skill params, mitigated by defender stats, with a crit roll. Enemies: idle until player in range, then approach and attack. Death → loot roll + XP. Keep AI simple: "walk toward player, attack when close" plus a few variants (ranged, fast melee, tanky brute).

### 5.3 Skills
v1 uses a **simpler model than POE2's socketed gems:** a small pool of unlockable active skills (melee strike, projectile, area attack), 2–3 assigned to hotkeys. Skills are **data-defined** (damage, cost, cooldown, area, projectile count). Each reads from the stat system so "+10% fire damage" from an item improves a fire skill. Gem system is a clean later upgrade.

### 5.4 Loot & items
On death: roll whether to drop, then rarity, base type, affixes. **Rarity:** Normal (no affixes), Magic (1–2), Rare (3–6), Unique later. **Affixes:** prefixes/suffixes from weighted pools by base type + level, each with tiers. Items live in an **inventory grid**, equip into slots (weapon, helmet, body, gloves, boots, ring×2, amulet); equipping recomputes stats. Heavily **data-driven** (base types, affix pools, drop tables = data, not code).

### 5.5 Passive skill tree
v1 ships a **small tree (~30–60 nodes)**: a point per level, spent to walk a connected graph of stat bonuses, with a few stronger "notable" nodes. Nodes just add modifiers to the same stat system — needs only a graph structure, allocation rules (allocate adjacent to owned), and UI. Reuses everything, so it's a great mid-project feature.

### 5.6 Progression
XP from kills → level up → passive point + small attribute gains. Zone/monster level scales difficulty and drop quality. v1 ceiling = zone boss + optional higher-difficulty replay.

## 6. Tech stack
**Unreal Engine 5.8 + C++** on **macOS** (Xcode 26.6 — [[Decisions]] ADR-006), built as a **3D top-down** ARPG (like the real Path of Exile). Chosen to play to the dev's strongest language (C/C++), match the POE reference with 3D's native strength, and use an industry-standard gameplay framework. Tradeoff explicitly accepted: Unreal is inheritance-heavy (`AActor`/`APawn`/`ACharacter`/components), which asks *more* OOP structuring than Godot would have — mitigated by leaning on Actor+Component composition and the guardrail docs. Full rationale and the superseded Godot decision are in [[Decisions]]. (Earlier evaluation had chosen Godot 4 + GDScript; libGDX, raw C++/Raylib, and Rust+Bevy were the other options considered.)

## 7. Architecture notes (fights "weak at OOP structuring")
- **Data-driven design.** One Item system and one Monster system reading specifics from data: ~~data files (Godot `Resource`/JSON)~~ **Unreal `UPrimaryDataAsset` / `UDataTable`** — stats, meshes, affix pools, drop tables. Adding content = editing data, not writing classes.
- **Composition over inheritance.** A game object = ~~a node with child components~~ **an Actor with components** (Health, Hitbox, AI), not a deep inheritance tree. When tempted to make a base class, ask "can this be a component?"
- **The stat aggregation system is the spine.** A `Stats` object holds base values + a list of modifiers; anything contributes modifiers; a recompute sums them. Build and test it first — items, passives, skills all become "things that add modifiers."

## 8. Build order (each step is playable)

> [!warning] **Superseded in v0.3** by milestones M0–M5 in [[Production-Plan-2026-09-14]]. Every step below is mapped into a milestone in [[Production-Plan-2026-09-14#2. How the old build order maps into milestones|Production Plan §2]]. Kept for history.
1. **Move & attack** — character in an empty level; learn Unreal's gameplay framework (Actor/Pawn/Controller/GameMode) + Enhanced Input.
2. **Enemies & combat** — dummy monsters with a Health component + trivial AI.
3. **Stat system** — build the modifier-aggregation spine; route numbers through it.
4. **Loot drops** — data-driven item + affix + rarity; items on the ground; pickup.
5. **Inventory & equipment** — grid inventory + equip slots feeding the stat system.
6. **Skills expansion** — 2–3 data-defined skills on hotkeys.
7. **Passive tree** — small node graph; point per level; allocation UI.
8. **Zone & boss** — real map, paced spawns, end boss, XP/leveling, difficulty toggle. **v1 done.**
9. **Polish** — hit feedback, sound, number formatting, loot filter, damage numbers, screen shake.

## 9. Immediate next actions

> [!warning] **Superseded in v0.3.** Current next actions live in [[Production-Plan-2026-09-14#M0 — Pre-production (paper only)|Production Plan M0]] and on [[HOME]].

~~Install Unreal Engine 5 + a C++ toolchain (Visual Studio / Xcode), then complete an official UE5 top-down or C++ tutorial to learn the gameplay framework.~~ Then build step 1 (move & attack) — see [[Step-1-Move-and-Attack]]. Sketch the core stats and two starter skills on paper so step 3 has a target. Pick a rough art direction (e.g. a free dark-fantasy 3D asset pack).
