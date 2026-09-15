---
title: Game Production — Research
date: 2026-09-14
status: research-draft
tags: [phoenix, research, production, planning]
---

# Game Production — Research

> [!info] **Scope.** How game teams structure planning: stages, milestones, prototyping. Then a
> line-by-line comparison against [[Design-Doc]]. **Facts and gaps only — no plan.** The plan is a
> separate paper, written after the decisions in §5 are made.

## Revision history

| rev | date | what changed |
|---|---|---|
| r1 | 2026-09-14 | First draft. |
| **r2** | **2026-09-15** | **§1 re-sourced after adversarial review.** r1's vertical-slice quote was not in any cited page; stage list and Alpha quote came from an uncited page (Kokku); First Playable definition omitted "polished … not a rough prototype". All quotes re-checked against page text. §1.1 split into two real passages. Greybox claim moved to §6. |

## Evidence tiers

| tag | meaning |
|---|---|
| **[source]** | Quoted from the Phoenix vault |
| **[web]** | Industry articles, fetched 2026-09-14. **Weaker than a book or first-party studio document** — most are studio or consultancy blogs |
| **[NOT verified]** | See §6 |

---

## 1. Stages of development

> [!warning] **Corrected in r2.** r1 quoted a "vertical slice" definition (*"maybe one level or a single mission
> … near-final quality"*) that is **not present in any page r1 cited**, and took the stage list and Alpha quote from a
> page r1 did not cite. Cause: a web-search **summary** was quoted as if the pages had been read. Every quote below
> was re-checked against the page text on 2026-09-15. ~~Struck r1 text is not reproduced; see revision history.~~

**[web]** Sources disagree on grouping and on what each name means. Each row names its own source.

| milestone | definition, quoted | source |
|---|---|---|
| Stages | "The eight stages are: concept, pre-production, vertical slice, production, alpha, beta, certification, and launch, with live ops extending the cycle well beyond ship date." | Kokku Games |
| **First playable** | "The milestone that ends Pre-Alpha is usually called 'First Playable' or 'Proof of Concept.' This is a **narrow, polished slice**, maybe 10-15 minutes of gameplay, that proves the core mechanic works and the visual style is achievable. … **Don't treat it as a rough prototype. Treat it like a pitch.**" | GameDevProducer |
| **Prototype** | "A prototype answers a focused question." | Playable Business (gamingindustry247) |
| **Vertical slice** | "A vertical slice is a representative playable segment intended to demonstrate how several disciplines and systems work together at a target or near-target quality bar. **Its exact scope varies.**" | Playable Business (gamingindustry247) |
| **Alpha** | "Alpha means every feature is functional and testable. UI, progression systems, combat, analytics, accessibility layers, all of it must be in the build. Content can be rough, but nothing can be missing." | Kokku Games |
| Beta / Gold | Beta: finalising content, assets and core functions, focusing on optimisation. Gold: preparing for publication. *(paraphrase)* | RocketBrush |

> [!important] **Delta against Phoenix's use of the terms.** GameDevProducer's "first playable" is *polished* and
> explicitly *not* a rough prototype. What Phoenix's production plan calls M1 is what Playable Business calls a
> **prototype**. The plan's naming must follow from this, not from "what the industry means" — sources do not agree.

### 1.1 The most important point in the sources

**[web]** Playable Business (gamingindustry247) — two separate passages, not one quote:

> "'Alpha,' 'beta,' 'content complete,' and 'release candidate' mean only what a team's written criteria say they
> mean."

> "… the labels are management tools, not universal gates. Each stage should have an owner, evidence, budget,
> risks, and an exit decision."

So what makes planning professional is not the vocabulary. It is that **every milestone has written exit
criteria** that can be checked, so "are we at alpha?" has a yes-or-no answer.

## 2. Prototype the core loop first

**[web]** Holly Green, *Game Developer*, 14 June 2022, quoting designer Dan Spaventa:

- A **core loop** is "the gameplay upon which the entire game is built": the actions repeated most often.
  RPGs typically: *Explore, Fight, Upgrade*.
- **Prototype the core loop first.** "If your core loop isn't fun, it doesn't matter how great your
  narrative or physics interaction or other features, if they are not having fun 'minute to minute',
  they are going to drop off and stop playing."
- **Every new system must strengthen the core loop.** Competing systems should be eliminated early.
- Prototyping is how teams "find the fun" and catch flaws **before** full production.

**[web]** A **greybox** prototype gets something playable in front of a tester using basic shapes (cubes,
spheres) instead of finished art. ~~Sources describe nearly every major studio doing some form of it.~~ *(r2: not found in any reachable cited page — moved to §6.)*

## 3. Phoenix's design doc, compared

Each row quotes [[Design-Doc]] directly.

### G1. The riskiest pillar is checked last

**[source]** Pillar 1, §2:
> **Satisfying combat.** Moment-to-moment play must feel good on its own — responsive movement, **clear
> hit feedback**, enemies that die in satisfying bursts.

**[source]** Build order, §8, final step:
> 9. **Polish** — **hit feedback**, sound, number formatting, loot filter, damage numbers, **screen shake**.

Hit feedback is part of what pillar 1 *means*, and it is scheduled after everything else. By §2's
standard, whether combat is satisfying is only answerable at step 9, after loot, inventory, skills,
passives and the boss are built on top of it. The industry practice is the reverse: prove the
minute-to-minute loop first.

### G2. "Vertical slice" names something bigger than a vertical slice

**[source]** §3:
> **Scope (v1 = vertical slice).** **In scope:** a single character …, click-to-move, one or two active
> skills, a single zone populated with several enemy kinds …, items with rarity tiers and randomized
> affixes, an inventory + equipment slots …, a small passive tree (a few dozen nodes) …, and an end boss.

Against §1: that list is **every v1 feature**, which is the sources' definition of **alpha** ("nothing can
be missing"). A vertical slice in the sources is a *small section at near-final quality*. Phoenix has no
milestone that is a small, polished proof, and the name it uses points at its largest one.

### G3. The pillars cannot be tested

**[source]** §2: "must feel good", "occasionally exciting", "noticeably different characters".

None of these has a criterion that lets a playtest pass or fail. Compare [[Step-1-Move-and-Attack]], whose
acceptance criteria are all objectively checkable.

### G4. Only step 1 of 9 has exit criteria

**[source]** `02-build-steps/` contains one spec, [[Step-1-Move-and-Attack]]. Steps 2–9 exist as one line
each in §8, with no definition of done.

### G5. No list of risks

The doc has no section naming what could sink the project. Candidates visible in this vault alone:
combat feel not reached (G1); Unreal learning curve; no art or animation source; scope creep (§3 calls the
scope line "the single most important thing for actually finishing"); macOS toolchain surprises (already
hit twice — see [[Toolchain-Research-2026-09-14]]).

### G6. Content is out of date

| where | says | now true |
|---|---|---|
| frontmatter | `version: 0.1 draft`, `updated: 2026-07-13` | body says "Version 0.2 draft · 2026-08-03" |
| frontmatter, header | `platform: PC / Desktop` | **macOS only** — [[Decisions]] ADR-006 |
| §7 | "data files (**Godot `Resource`**/JSON)", "a node with child components" | Unreal: `UDataAsset` / `UDataTable`, Actor + Components |
| §9 | "**Install Unreal Engine 5** + a C++ toolchain (Visual Studio / Xcode)" | Installed and verified: UE 5.8.2 + Xcode 26.6 |

### G7. Loot has structure but no numbers

**[source]** §5.4 defines the shape of loot ("weighted pools", "tiers", rarity bands) but no drop rates,
weights or tier ranges. Pillar 2 ("frequent … occasionally exciting") is decided by exactly those numbers.

### G8. No capacity, estimates or task tracking

No hours per week, no estimate per step, no backlog. Without capacity, milestones cannot have dates.

## 4. What the doc already does well

- **Pillars exist, and the doc uses them to cut scope:** "Anything that doesn't serve one of these three is
  out of scope for v1."
- **Explicit out-of-scope list** in §3.
- **Core loop stated** in §4, at two time scales.
- **Data-driven design** chosen from the start, which is what makes tuning (G7) possible later.
- **Step 1 is a model spec**, with acceptance criteria and a learning checklist.

## 5. Decisions this research surfaces (for the plan, not decided here)

1. Adopt stage-based milestones (greybox prototype → vertical slice → alpha → beta), each with written exit
   criteria, in place of the flat nine-step order.
2. Move combat feel (hit feedback) into the first prototype — G1.
3. What "v1" is called, given G2. **Renaming an established term needs approval.**
4. Weekly capacity, so milestones can have dates — G8.

## 6. NOT verified

| claim | why | what would settle it |
|---|---|---|
| Any definition in §1 is *the* industry meaning | Sources are studio and consultancy blogs and **disagree** (§1 delta) | A production text such as Heather Chandler's *The Game Production Handbook*, or a first-party studio document |
| "Nearly every major studio" greyboxes | Came from a search summary; the cited page (maxcomperatore) did not resolve, and mgackowski does not say it | A reachable source that states it |
| ~~The "written criteria" wording~~ | **Resolved r2** — present on the Playable Business page (§1.1) | — |
| ~~GameDevProducer milestone article unreadable (HTTP 403)~~ | **Resolved r2** — the 403 was specific to one fetch tool; the page loads and is quoted in §1 | — |

## Sources

- Kokku Games — [What are the essential stages of full cycle game creation](https://kokkugames.com/what-are-the-essential-stages-of-full-cycle-game-creation/) *(added r2)*
- GameDevProducer — [Alpha, Beta, Gold: The Stages Every Game Needs](https://gamedevproducer.com/posts/what-is-a-game-milestone-alpha-beta-gold/) *(added r2)*

- Holly Green — [How supporting core loops and early prototyping are key to your game's success](https://www.gamedeveloper.com/design/how-supporting-core-loops-and-early-prototyping-are-key-to-your-game-s-success), *Game Developer*, 2022-06-14
- [Game development stages: full cycle from idea to launch](https://game-ace.com/blog/game-development-stages/)
- [Game Development Process: Key Phases and Insights](https://rocketbrush.com/blog/game-development-process-guide)
- [Game Development Stages: From Pitch to Live Support](https://gamingindustry247.com/game-development-stages/)
- [Why Greyboxing and Prototypes are Vital in Game Development](https://blog.maxcomperatore.com/why-greyboxing-and-prototypes-are-vital-in-game-development)
- [Grey box prototyping — inverted ↄontrols](https://mgackowski.wordpress.com/2021/02/16/grey-box-prototyping/)

## Related

- [[Design-Doc]] · [[00-Research-Hub]] · [[Documentation-Framework]]
