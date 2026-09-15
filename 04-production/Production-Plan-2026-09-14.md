---
title: Production Plan
date: 2026-09-14
status: draft-awaiting-comment-step
tags: [phoenix, plan, production, milestones]
---

# Phoenix — Production Plan

> [!info] **What this is.** How Phoenix gets from an empty folder to v1, organised as **milestones with
> written exit criteria**, not a feature list. Built on [[Game-Production-Research-2026-09-14]] (numbering
> such as "G1" and "§1.1" in this note refers to that research paper unless stated).
> **Planning only — no game code is written under this paper until the user says "go build".**

## Revision history

| rev | date | what changed |
|---|---|---|
| r1 | 2026-09-14 | First draft. Decisions taken: adopt milestone stages; rename v1 to Alpha; tracking in Obsidian. |

---

## 1. Summary

```
M0 Pre-production ──► M1 First Playable ──► M2 Vertical Slice ──► M3 Alpha (= v1 features) ──► M4 Beta ──► M5 v1 Release
   (paper)              (greybox combat)       (small, polished)       (everything works)          (complete,     (packaged,
                                                                                                    tuned)         playable)
```

> [!note] Readable copy on the [Phoenix Miro board](https://miro.com/app/board/uXjVHndMfFs=/?moveToWidget=3458764683756097734); **this note is canonical.**

- **Every milestone ends on a yes/no checklist** (§3). "Are we at M2?" is answered by ticking boxes, not by feel.
- **Combat feel is proven first**, in M1, before loot, inventory, skills or passives exist (fixes G1).
- **"Vertical slice" now means what the industry means**: a small section at near-final quality (M2). The old
  §3 scope of [[Design-Doc]] is **M3 Alpha** (fixes G2).
- **No dates.** Milestones are scope-and-criteria driven. Real hours are logged per session so the M2 estimate
  can be based on measured M1 pace (fixes G8 honestly).

## 2. How the old build order maps into milestones

Nothing from [[Design-Doc]] §8 is dropped. Every step lands in a milestone, and some are split because a small
version is needed earlier.

| old §8 step | goes to | what changes |
|---|---|---|
| 1. Move & attack | **M1** | unchanged scope |
| 2. Enemies & combat | **M1** (dummy pack) → **M2** (2 enemy types + AI variants) → **M3** (all enemy kinds) | split |
| 3. Stat system | **M2** | needed once numbers vary; M1 uses fixed constants |
| 4. Loot drops | **M1** (placeholder cube drops) → **M2** (rarity + small affix pool) → **M3** (full pools) | split |
| 5. Inventory & equipment | **M2** (2 slots, no grid) → **M3** (grid + all slots) | split |
| 6. Skills expansion | **M2** (2 skills) → **M3** (full v1 set) | split |
| 7. Passive tree | **M2** (~8 nodes) → **M3** (30–60 nodes) | split |
| 8. Zone & boss | **M2** (one encounter area + mini-boss) → **M3** (full zone + boss) | split |
| 9. Polish | **M1** (hit feedback — pillar 1) → **M2** (near-final feel, sound) → **M4** (everything else) | **moved earlier** — this is G1 |

## 3. Milestones and exit criteria

### M0 — Pre-production (paper only)

**Purpose:** everything M1 needs to start without guessing.

**Exit criteria**
- [ ] This plan passes the Comment step (user read + adversarial review) and is marked `approved`
- [ ] [[Design-Doc]] updated to v0.3 — G2 rename, G6 stale content
- [x] Toolchain proven: UE 5.8.2 + Xcode 26.6 compile and editor start — [[Toolchain-Research-2026-09-14]]
- [ ] **R2** Unreal C++ project anatomy — research paper
- [ ] **R5** Game feel techniques — research paper (feeds M1 exit criteria)
- [ ] **R3** Asset naming + `Content/` layout — research paper (before the first asset exists)
- [ ] Glossary started, verified terms only
- [ ] **M1 spec** written as a build-step note, replacing the current Step-1 spec's scope with M1's

### M1 — First Playable: greybox combat prototype

**Purpose:** answer one question — **does hitting things feel good?** — as cheaply as possible.

**Scope**
- One greybox level (floor + a few blocks), fixed top-down camera
- Click-to-move with pathfinding; one basic attack
- A pack of 5–10 **cube** enemies: health, walk toward player, die
- **Hit feedback set** (exact techniques chosen from R5): flash on hit, a brief pause on impact, knockback,
  floating damage number, death burst
- Enemy death drops a placeholder cube that can be picked up (logs "picked up") — proves the loop closes
- Fixed numbers in code or a data asset; **no stat system yet**

**Out of scope:** art, animation, sound, UI beyond damage numbers, stats, real loot, skills, save/load.

**Exit criteria — objective**
- [ ] A fresh clone builds and opens with no errors on this Mac
- [ ] Click-to-move reaches any reachable floor point and stops cleanly
- [ ] Attack damages only enemies in range; enemies die at 0 health
- [ ] Every item of the hit feedback set is visible on every hit, and each can be switched off individually (for comparison)
- [ ] Clearing a pack of 5 cube enemies works end to end without errors in the Output Log
- [ ] Death spawns a pickup; walking over it logs the pickup
- [ ] All feedback timings live in one tunable data asset, not scattered constants

**Exit criteria — playtest (pillar 1)**
- [ ] **A/B test:** 3 people (including the developer) play with all hit feedback **off**, then **on**. All 3 prefer "on" and can say why
- [ ] Each tester clears 3 packs in a row and, asked "would you keep doing this for 5 more minutes with no rewards?", at least 2 of 3 say yes
- [ ] Findings written into the Build-Log, including what felt wrong

> [!warning] The playtest thresholds (3 people, 2 of 3) are **starting hypotheses set by this plan**, not
> industry figures. Revise them after M1 if they prove too weak or too strict.

### M2 — Vertical Slice

**Purpose:** prove **all three pillars together**, at near-final quality, in a small space.

**Scope**
- One encounter area (not a full zone), lit and dressed with a free asset pack
- Player character with a real mesh and basic animation set
- 2 skills (one melee, one projectile) on hotkeys
- 2 enemy types with different behaviour + 1 mini-boss
- **Stat system** (the spine) rebuilt, all numbers routed through it
- **Loot:** Normal/Magic/Rare rarity, a small affix pool (~10 affixes), weighted rolls from a **tuning data table**
- Pick up → equip into **2 slots** (weapon, body armour) → stats change visibly
- **~8-node passive tree** with at least 2 branches that change how the character plays
- Sound on hit, death, loot drop

**Exit criteria — objective**
- [ ] Stat aggregation has automation tests, proven to fail when the aggregation is broken
- [ ] Loot rolls are reproducible from a seed (for testing and tuning)
- [ ] Drop rates, affix weights and tier ranges live in a data table, not in code (fixes G7)
- [ ] Equipping and unequipping updates stats with no leftover modifiers
- [ ] The slice plays start to finish (enter area → clear → mini-boss → end) with no errors
- [ ] Stable frame rate on this Mac at the editor's default quality (target set after M1 measurement)

**Exit criteria — playtest (all pillars)**
- [ ] **Pillar 1** — M1's A/B result still holds with real art and animation
- [ ] **Pillar 2** — in a 10-minute session, each tester can name at least one drop that excited them
- [ ] **Pillar 3** — two testers who pick different passive branches can each describe how their character plays differently

### M3 — Alpha (= v1 features, formerly "v1 = vertical slice")

**Purpose:** every v1 feature from [[Design-Doc]] §3 exists and works. Content can be rough; **nothing is missing**.

**Exit criteria**
- [ ] Every "In scope" item in [[Design-Doc]] §3 is functional and testable
- [ ] Full zone with paced spawns; end boss; XP and levelling; difficulty toggle
- [ ] Grid inventory + all 8 equipment slots
- [ ] Passive tree with 30–60 nodes, including notables
- [ ] Full v1 skill set
- [ ] No known crash bugs

### M4 — Beta

**Purpose:** content complete, tuned, polished.

**Exit criteria**
- [ ] All planned content present — no placeholders
- [ ] Loot and difficulty tuned from playtest data
- [ ] Remaining polish from old step 9: number formatting, loot filter, screen shake where missing
- [ ] Performance target met on this Mac
- [ ] A full playthrough by someone who has not played before, without help, reaches the boss

### M5 — v1 Release

**Exit criteria**
- [ ] Packaged, standalone macOS `.app` runs outside the editor on this Mac
- [ ] Git tag `v1.0.0` on the commit that produced it
- [ ] Postmortem written: what worked, what did not, what the engine project should learn

## 4. Pillars made testable (fixes G3)

| pillar | first proven in | test |
|---|---|---|
| 1. Satisfying combat | M1 | Feedback on/off A/B preference; "5 more minutes?" retention question |
| 2. Meaningful loot | M2 | Each tester names an exciting drop within 10 minutes |
| 3. Build expression | M2 | Two testers on different branches describe different playstyles |

Proposed replacements for the untestable wording in [[Design-Doc]] §2 go into v0.3 of that note, pointing here.

## 5. Risk register (fixes G5)

| # | risk | likelihood | impact | mitigation | checked at |
|---|---|---|---|---|---|
| K1 | Combat never feels good | medium | **fatal** to pillar 1 | M1 exists only to test it; R5 before M1 | M1 exit |
| K2 | Unreal learning curve stalls progress | high | high | Research papers per concept; M1 deliberately small | every session log |
| K3 | No art or animation source for M2 | medium | high | Research free asset packs before M2; greybox until then | before M2 |
| K4 | Scope creep | high | high | Out-of-scope lists per milestone; new ideas go to a parking list, never into the current milestone | every milestone |
| K5 | macOS toolchain surprises | medium (hit twice) | medium | Record each in the toolchain research; the fallback is Xcode 26.1.1 | first compile of Phoenix |
| K6 | Shader Model 5 fallback blocks a needed feature | low | medium | Note which features need SM6 if one is wanted | when a rendering feature is chosen |
| K7 | Motivation drops in long infrastructure stretches | medium | high | Every milestone ends in something playable | every milestone |

## 6. Research schedule

Each item lands in `03-research/` before the milestone that depends on it.

| # | topic | needed before |
|---|---|---|
| R2 | Unreal C++ project anatomy (`.uproject`, targets, modules, UnrealHeaderTool) | M1 |
| R3 | Asset naming conventions + `Content/` folder layout | M1 (first asset) |
| R4 | nvim + clangd workflow — a How-to guide, not a research paper | M1 |
| R5 | **Game feel techniques** (hit feedback: what each technique is, typical use) | **M1** |
| R6 | Loot table design (weighting, rarity, affix tiers) | M2 |
| R7 | Stat and modifier systems; custom component vs Gameplay Ability System | M2 |
| R8 | Asset sources: free character, animation and environment packs | M2 |
| R9 | Unreal automation testing | M2 |

## 7. Capacity, cadence and roles

- **No calendar dates.** Progress is measured by exit criteria ticked.
- **Every work session** gets a Build-Log line with hours spent. After M1, the M2 estimate is made from real
  numbers.
- **Roles**

| who | does |
|---|---|
| **User** | Decides; hand-writes game code (per [[Decisions]] ADR-003); clicks in the Unreal editor; playtests; says "go build" |
| **Claude** | Research papers, plan papers and specs; reviews; verifies builds and logs; explains each file before it is written |

Claude can keep working on **paper items** while the user is away, as long as the session stays open and the
Mac is awake. Anything needing the user's hands, eyes or approval waits.

## 8. Tracking

Obsidian only, during planning. Each milestone's exit criteria are its checklist. Current milestone and next
action are always visible on [[HOME]].

## 9. Alternatives considered

| alternative | rejected because |
|---|---|
| Keep the flat 9-step order | Combat feel only testable at step 9 (G1); no milestone is a small polished proof (G2) |
| Build every feature first, polish last | Same as above; the pillars would be validated after the dependencies are fixed |
| Skip the vertical slice, go straight to Alpha | Loses the only milestone that proves all three pillars together before full content is built |
| Calendar-dated milestones | No measured pace yet; dates now would be guesses. Revisit after M1 |

## 10. Not applicable to this paper

The house plan template includes an error-contract table and a state-integrity table. Both concern code that
fails or mutates state; a production plan does neither. They return in the M1 spec, where code exists.

**Rollback:** this paper is text. Reverting means restoring the previous revision; nothing downstream depends on it
until it is `approved`.

## 11. Open questions

- None blocking M0. Playtest thresholds in M1/M2 are hypotheses to revisit after M1 (§3 warning).

## Related

- [[Game-Production-Research-2026-09-14]] · [[Design-Doc]] · [[Decisions]] · [[HOME]]
