---
title: Combat — Game Design
date: 2026-09-15
status: draft-awaiting-comment-step
revision: r2
feature: combat
pillar: 1 — Satisfying combat
tags: [phoenix, game-design, combat, pilot]
---

# Combat — Game Design

> [!info] **What this is.** What the player experiences when fighting: rules, numbers, feedback, edge cases. **Pilot
> spec:** the first feature written in full, to test the feature-spec template before the other fifteen. How it is
> built lives in [[Combat-Tech]].
>
> Every **[design]** item is a proposal open to review. Every number marked **(H)** is a **starting hypothesis to
> tune in playtest**, not a known-good value.

## Revision history

| rev | date | what changed |
|---|---|---|
| r1 | 2026-09-15 | First draft — pilot of the feature-spec template. |
| **r2** | **2026-09-15** | After two independent reviews (design + claim verification), all findings re-checked against engine source. See **§15** for every change and why. |

---

## 1. Purpose

**Pillar served:** 1 — *Satisfying combat*. [[Design-Doc#2. Vision & pillars|Design-Doc §2]]:

> "Moment-to-moment play must feel good on its own — responsive movement, clear hit feedback, enemies that die in
> satisfying bursts."

Combat is the action the player repeats most. [[Design-Doc#4. Core gameplay loop|Design-Doc §4]] core loop: *"move →
engage a pack → use skills to kill → collect loot → repeat."*

**This spec covers:** walking up to an enemy, the basic attack, damage, health, knockback, death (enemy and player),
and hit feedback. **Skills** reuse these rules and get their own spec.

## 2. Player experience

In order of importance [design]:

1. **I clicked, it happened.** The character responds to the most recent input immediately.
2. **I can see and hear every hit land**, even inside a pack of ten.
3. **Big hits feel big.** Crits and killing blows read differently from normal hits.
4. **Packs die in bursts.** Several deaths close together feel like a payoff, not a to-do list.
5. **I know when I'm in danger.** Taking damage is as readable as dealing it.

## 3. Controls

[design], following [[Design-Doc#5.2 Combat|Design-Doc §5.2]] (*"Left-click moves/attacks; number/letter keys trigger
skills."*):

| input | on | result |
|---|---|---|
| Left-click | ground | Move to the point |
| Left-click | enemy | Walk into range, then attack it |
| Hold left mouse | enemy | Keep attacking that enemy until it dies or the button is released |
| Hold left mouse | ground | Keep moving toward the cursor |
| Number keys 1–3 | anywhere | Skills — see the Skills spec |

**Which enemy is "under the cursor":** of all enemies whose capsule lies within **40 cm (H)** of the point under the
cursor, the one nearest to that point.

> [!warning] **Conflict resolved here.** [[Step-1-Move-and-Attack]] maps attack to **Right Mouse Button**,
> contradicting [[Design-Doc#5.2 Combat|Design-Doc §5.2]]. This spec follows the Design Doc.

## 4. The basic attack

### 4.1 Phases

```
 click enemy ─► APPROACH ────► WINDUP ─────► IMPACT ─► RECOVERY ─────► ready
                (walk until     (anticipation;  (damage   (follow-through;
                 in range)       can't move)     here)     input buffered)
```

| value | starting value (H) | note |
|---|---|---|
| Windup | 0.15 s | Short enough to feel responsive; long enough to read |
| Recovery | 0.25 s | Stops attacks blurring together |
| Attacks per second | 2.5 | = 1 / (0.15 + 0.25) |
| Melee range | 180 cm | **Measured capsule surface to capsule surface, ignoring height** |
| Approach stops at | melee range − 30 cm | So a target stepping back during windup is still in range |
| Approach gives up after | 3 s | Target unreachable or kiting |

### 4.2 Rules

1. **Single target.** The basic attack hits only the chosen enemy. See open question 3 — this may change. [design]
2. **An attack cannot start on a dead or dying enemy.** Clicking one does nothing.
3. **Damage lands at impact.** If the target died **after windup began**, or is out of range at impact, the attack
   **misses** and shows a whiff. [design]
4. **Ground click cancels.** A ground click during approach or windup cancels the attack with no damage.
5. **Input during windup and recovery.** The **latest** click is kept and runs when recovery ends. A ground click
   during recovery ends recovery immediately and moves. A kept attack on a target that has since died is dropped.
   [design]
6. **Can't attack while dead.**
7. **Attack speed (from M2).** Windup and recovery are divided by (1 + attack speed modifier).

### 4.3 Knockback — a gameplay rule, not a visual

**[design]** Only a **killing blow** knocks the enemy back (distance **(H)**). Normal hits do not move the target.

**Why not every hit:** pushing the target on every hit moves it out of melee range, so the next impact whiffs and
hold-to-attack becomes hit, miss, walk, hit. Knockback moves enemies, which changes whether later attacks land, so it
is a **gameplay rule** and **cannot** be part of the switchable feedback set in §7.

## 5. Damage

### 5.1 Damage types and teams

- [[Design-Doc#5.1 Character & stats|Design-Doc §5.1]]: *"Damage split into Physical/Fire/Cold/Lightning."* One hit may
  carry several types.
- **Teams:** every character is on team **Players** or team **Monsters**. Attacks only damage the other team — there
  is no friendly fire. [design]

### 5.2 The formula

**[design]** In this exact order:

```
0. crit   = ONE roll per hit:  roll < attacker's critical strike chance      (roll uniform in [0, 1))
1. for each damage type whose raw damage is greater than 0     (types with no raw damage are skipped entirely)
     raw'   = crit ? raw × critical strike multiplier : raw
     Physical:             a         = max(Armour, 0)
                           reduction = min( a / (a + ArmourK × raw') ,  90 % )
     Fire / Cold / Lightning:
                           reduction = min( that type's own resistance ,  75 % )
     taken  = raw' × (1 − reduction)
2. final  = sum of taken over all types, then rounded half up (2.5 → 3)
3. if any type had raw damage > 0:  final = max(final, 1)
```

| constant | starting value (H) | effect |
|---|---|---|
| Critical strike chance (base) | 5 % | |
| Critical strike multiplier (base) | 150 % | |
| `ArmourK` | 5 | Higher = armour weaker against big hits |
| Armour reduction cap | 90 % | |
| Resistance cap | 75 % | |

**Why armour has this shape:** reduction shrinks as the *hit* grows. Armour protects well against many small hits and
poorly against one huge hit, so big hits stay dangerous and armour never makes anyone immune.

**Why one crit roll per hit:** a hit either crits or it doesn't — the player sees one crit number and one crit sound.

**Why skip types with no raw damage:** otherwise a pure fire hit against 0 armour computes `0 / (0 + 5 × 0)` for its
physical part, which is undefined.

**Why "minimum 1":** a hit that visibly lands but shows 0 reads as a bug.

### 5.3 What M1 uses

M1 has **no stat system**. Every number above comes from tuning data; armour and resistances are 0. The formula's
structure is identical, so M2 swaps tuning values for stats without changing rules.

## 6. Health and death

### 6.1 Health

- Health never goes below 0 or above maximum.
- Enemies do not regenerate. Player regeneration is a stat (Stats spec).

### 6.2 Enemy death

**[design]** At 0 health, exactly once:

1. The enemy stops acting and stops blocking movement immediately.
2. The killing blow's knockback applies (§4.3). **Death feedback** plays (§7).
3. **Loot and XP are granted** — combat only announces the death and who killed it (Loot, Progression specs).
4. The body stays for **3 s (H)**, then disappears.

A player who arrives near a body later **does not** see the death burst again.

### 6.3 Player death

**[design]** — **gap in [[Design-Doc]]**, proposed for v1:

- Controls lock; death feedback plays. A dead player grants **no** loot or XP to anyone.
- After **3 s (H)**, the player respawns at the zone entrance with full health.
- Enemies keep their current health and positions. No XP or item penalty in v1.

## 7. Hit feedback — visual and audio only

**The core of pillar 1.** Final set chosen by research item **R5** (game feel). Every item here is **purely visual or
audio** — switching one off must not change what happens in the game, only how it looks or sounds.

| technique | when | purpose |
|---|---|---|
| **Hit flash** | Every hit | "That hit landed" |
| **Hit-stop** — attacker's and target's **animation** freezes briefly; the game keeps running | Every hit; longer on crits and kills | Weight |
| **Damage number** | Every hit; crits larger and a different colour; shows the damage **dealt** (§10 overkill) | Exact information |
| **Hit sound** | Every hit; distinct crit and kill sounds | Impact without looking |
| **Screen shake** | The **attacking player's own** crits and kills only | Big moments feel big |
| **Death burst** | Enemy death; deaths within **0.3 s (H)** of each other share one amplified sound | Makes pack clears feel like bursts |
| **Whiff** | Attack misses (§4.2 rule 3) | Explains why no damage happened |
| **Windup tell** | Attack starts; in M1 a squash-and-stretch on the cube | Anticipation without animation |
| **Player damaged** | Player takes a hit | Screen-edge flash + sound |

**Rule:** every technique can be switched off individually and its strength is a tuning value. Without that,
playtesting cannot tell which one matters.

## 8. Multiplayer

**Decision context:** Phoenix v1 is single-player, built to be playable with several players. Stated in chat on
2026-09-15 as "option B". **Not yet recorded in [[Decisions]]**, and [[Design-Doc#3. Scope (v1 = Alpha: every v1 feature functional)|Design-Doc §3]]
still lists multiplayer as out of scope — both are resolved by an ADR once the scope question in §14 is answered.

| question | answer [design] |
|---|---|
| Can players hurt each other? | **No.** Teams (§5.1). |
| Whose damage numbers do I see? | **Mine at full size; other players' smaller and faded.** |
| Who gets the kill? | **The player whose hit killed it.** Sharing loot and XP is decided in the Loot and Progression specs. |
| Does hit-stop freeze anyone's game? | **No** — it freezes animation only (§7). |
| Does another player's crit shake my screen? | **No** (§7). |
| What if my hit arrives late? | The server decides damage. My **own** windup starts on my screen immediately; the damage result may appear a moment later. |

## 9. Tuning values

Grouped by **why they would change** [design]:

| group | holds | one copy or many |
|---|---|---|
| **Damage rules** | `ArmourK`, armour cap, resistance cap | One, game-wide |
| **Attack profile** | Windup, recovery, melee range, approach stop and give-up, base crit chance and multiplier, kill knockback | One per character type — a brute and a fast melee enemy differ |
| **Feedback tuning** | On/off and strength of each §7 technique, burst window | One |
| **Game rules** | Body linger time, respawn delay, cursor pick radius | One, game-wide |

## 10. Edge cases

| case | behaviour [design] |
|---|---|
| Target dies during approach | Attack cancelled; nothing shown |
| Target dies during windup (killed by someone else) | Miss at impact; whiff; a kept click on that target is dropped |
| Target steps out of range during windup | Miss at impact; the attacker does **not** chase mid-windup |
| Target unreachable, or kiting, during approach | Give up after 3 s (H) |
| Attacker dies during approach or windup | Attack cancelled; no damage |
| Two hits kill the same enemy in the same moment | Death happens once; kill credit to the hit the server applied first |
| Click an enemy behind a wall | Walk along the path into range, then attack |
| Overkill (hit for 500 on 20 health) | Damage number shows **500**; health stops at 0 |
| Hit on a hit rounded to 0 after mitigation (e.g. fire 1 at 75 % resistance) | Still minimum 1; still shown as a crit if it crit |
| Several enemies near the cursor | Pick rule in §3 |
| Player clicks another player | Treated as a ground click — move there |

## 11. Dependencies

| depends on | for |
|---|---|
| Player & camera, Input | Clicks, walking into range |
| Stats & modifiers | Damage, crit, armour, resistances, attack speed (from M2) |
| Enemies & AI | Enemies attack using these same rules |
| UI/HUD | Damage numbers, health bars |
| Audio & feedback | Hit and death sounds |
| **Used by:** Loot, Progression | Enemy death announcement with killer |
| **Used by:** Skills | Damage formula, hit feedback, teams |

## 12. Out of scope

Blocking, dodging, stagger or stun, status effects (burn, freeze, shock), damage over time, lifesteal, friendly fire,
gameplay prediction on the client (only the player's own windup animation starts early — §8), PvP.

## 13. Acceptance tests

Each can fail, and each says what it measures:

- [ ] Clicking an enemy 600 cm away walks into range and deals damage **exactly once**, without a second click
- [ ] Impact log time − windup start log time = windup ± 1 frame
- [ ] A target killed by another source during windup produces a miss and no damage
- [ ] A ground click during windup cancels with no damage dealt
- [ ] Holding on a stationary 10-hit enemy produces 10 impacts and 0 whiffs
- [ ] With one click kept during recovery, it runs when recovery ends; a second click replaces it
- [ ] The damage formula returns every value in [[Combat-Tech#11.1 Damage formula tests|Combat-Tech §11.1]]
- [ ] An enemy's death is announced **exactly once**, even when two hits land together
- [ ] Overkill shows the dealt number, not the health lost
- [ ] Each §7 technique increments its own counter; with technique X off, only X's counter stays 0 over a 5-kill run
- [ ] A dead player respawns at the zone entrance with full health after the respawn delay
- [ ] A player attacking another player deals 0 damage; the same attack on an enemy deals damage
- [ ] Every multiplayer row in [[Combat-Tech#11.3 Multiplayer matrix|Combat-Tech §11.3]] passes in all three modes

Pillar playtests are defined in [[Production-Plan-2026-09-14]] (§3); its thresholds are hypotheses to revise after M1.

## 14. Open questions

1. **Multiplayer scope for v1** — can a friend **actually join and play** in v1, or does v1 ship single-player with the
   structure ready? Decides whether [[Combat-Tech#11.3 Multiplayer matrix|Combat-Tech §11.3]] is a release gate.
2. **Player death rules** (§6.3) — respawn at the zone entrance with no penalty?
3. **Single target or cleave** (§4.2 rule 1) — the reviewers point out that a single-target attack at 2.5 hits per
   second, with no skills in M1, kills a pack **one at a time** — the "to-do list" §2.4 rejects. A small cleave (e.g.
   a 90° arc, 50 % damage to other targets **(H)**) would let M1's playtest test "packs die in bursts". **Decide before
   M1.**
4. **Starting values** — all (H) values are placeholders until M1's playtest.

## 15. Changes from r1

Each r1 claim that changed, why, and the evidence. r1 was not read by the user before revision.

| r1 said | r2 says | why |
|---|---|---|
| Click an enemy → "Move into range, then attack" with no phase for it | **APPROACH** phase added (§4.1) | Design review: no mechanism moved the character; see [[Combat-Tech]] §5.1 |
| Crit rolled inside the per-type loop | **One roll per hit** (§5.2 step 0) | Design review: a hit could crit on one type and not another, but there is one crit flag and one crit number |
| Formula computed physical for every hit | **Types with no raw damage skipped**; armour clamped ≥ 0 | Design review: pure elemental hit vs 0 armour computes 0 / 0; recomputed by script — seven tests become undefined without the skip |
| "rounded to the nearest whole number" | **Rounded half up** | Claim review: the test table depended on a tie rule the spec didn't state |
| Knockback in the switchable feedback set, on every hit | **Killing blow only, a gameplay rule** (§4.3) | Design review: it moves targets out of range and changes the game when switched off |
| Hit-stop "a very short freeze of attacker and target" | **Animation freeze only; the game keeps running** | Design and claim reviews: time dilation slows the real simulation on the machine running the server — `Actor.cpp:379`, `Actor.h:4890-4891` |
| "Melee range 180 cm — roughly one character-width past contact" | **Measured capsule surface to surface**; the width claim removed | Claim review: the default capsule radius is 34 cm, so 180 cm is about 2.6 widths |
| "No friendly fire" with nothing defining sides | **Teams: Players and Monsters** (§5.1) | Design review: nothing stopped a client attacking another player |
| Buffering "during recovery" only; §10 cleared a buffer during windup | **Latest click kept during windup and recovery**; one rule (§4.2 rule 5) | Both reviews: the two sections contradicted each other |
| "no input is swallowed" | **"responds to the most recent input"** | Design review: contradicted "more clicks overwrite" |
| Dead target at impact → whiff, yet "hit while dying → ignored" | **Can't start on a dead target; whiff only if it died after windup began** | Design review: contradiction |
| "Crit on a hit fully blocked by resistance cap" | **Hit rounded to 0 after mitigation** | Claim review: the 75 % cap makes a full block impossible |
| "a single tuning data asset" | **Four groups by reason to change** (§9) | Design review: a brute and a fast melee enemy need different timings; damage rules must be one global copy |
| Option B cited to [[Decisions]] | **Chat decision, not yet an ADR; conflicts with Design-Doc §3** | Claim review: no such entry exists |
| "In Play As Listen Server with 2 clients" | **Three-mode matrix** in Combat-Tech §11.3 | Claim review: 2 means host + 1 client (`PlayLevel.cpp:2893-2918`) |
| Acceptance tests without tolerances | **Tolerances and counters** (§13) | Design review: "at impact time" and "removes only that technique" could not fail objectively |
| No test for player death, hold, buffering, overkill, teams | **Added** (§13) | Design review |

## Related

- [[Combat-Tech]] · [[OOP-Foundations]] · [[Design-Doc]] · [[Production-Plan-2026-09-14]]
