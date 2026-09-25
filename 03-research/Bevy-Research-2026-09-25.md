---
title: Bevy — Research
date: 2026-09-25
status: research-draft
tags: [phoenix, research, rust, bevy, engine]
---

# Bevy — Research

> [!info] **Scope.** What Bevy is today, and what moving Phoenix from Unreal to Rust + Bevy would mean.
> **Facts only — the decision is [[Decisions]] ADR-008.** Written after the user asked to switch (2026-09-25).

## Revision history

| rev | date | what changed |
|---|---|---|
| r1 | 2026-09-25 | First draft, from bevy.org and docs.rs. |

## Evidence tiers

| tag | meaning |
|---|---|
| **[doc]** | bevy.org or docs.rs, page text checked 2026-09-25 |
| **[NOT verified]** | See §5 |

---

## 1. What Bevy is

**[doc]** A data-driven game engine written in Rust, built on **ECS** (entity–component–system): data lives in
components, behaviour lives in systems, and the engine schedules those systems. Composition is the default; there is
no class hierarchy to inherit from.

**Releases** (bevy.org/news):

| version | date |
|---|---|
| **0.19** | 19 June 2026 (current) |
| 0.18 | 13 January 2026 |
| 0.17 | 30 September 2025 |
| 0.16 | 24 April 2025 |

## 2. The two facts that matter most

**[doc]** bevy.org, *Migration Guides* introduction:

> "Bevy is still in the 'experimentation phase', which means each release has its fair share of breaking changes."

> "We provide migration guides for major releases to help users migrate their apps to the latest and greatest Bevy
> release."

Sixteen migration guides are listed, from 0.4→0.5 up to 0.19→0.20. **Every release so far has broken APIs**, roughly
every 4–5 months.

**[doc]** There is **no editor yet**. The 0.19 release post refers to *"the upcoming Bevy Editor"* and to an
*"eventual editor"*. Everything — levels, entity placement, tuning — is code or data files you write, not a scene
editor you click in.

## 3. The ecosystem an action RPG needs

**[doc]** All three are current and target Bevy 0.19:

| need | crate | version, date | description (quoted) |
|---|---|---|---|
| Physics and collision | `avian3d` | 0.7.0, 2026-06-20 | "An ECS-driven physics engine for the Bevy game engine" |
| Pathfinding for click-to-move | `vleue_navigator` | 0.16.0, 2026-08-25 | "Navigation mesh for Bevy using Polyanya" |
| Co-op networking | `bevy_replicon` | 0.44.2, 2026-09-22 | "A server-authoritative replication crate for Bevy" |

`bevy_replicon` being **server-authoritative** matches the model already chosen for Phoenix
([[OOP-Foundations#7. Multiplayer-ready objects (decision: option B)|OOP-Foundations §7]], ADR-007) — the principle
carries over even though the Unreal mechanics do not.

## 4. What this costs and what it keeps

**Keeps — engine-independent:**

| note | why it survives |
|---|---|
| [[Design-Doc]] | pillars, scope, core loop, systems — about the game, not the engine |
| [[Combat]] | rules, damage formula, feedback, edge cases, acceptance tests |
| [[Production-Plan-2026-09-14]] | milestones and exit criteria |
| [[Documentation-Framework]], [[Decisions]], [[Build-Log]] | process and history |
| [[Game-Production-Research-2026-09-14]] | how studios plan |

**Retires — Unreal-specific:**

| note | why |
|---|---|
| [[Combat-Tech]] | classes, RPCs, replication, PIE test matrix are Unreal mechanisms |
| [[Network-Protocol]] | built on Unreal RPC and replication semantics |
| [[OOP-Foundations]] | Unreal's UObject, inheritance, ownership; Bevy has no class hierarchy — the *reasoning* about responsibilities and authority survives, the mechanics do not |
| [[Architecture]], [[Coding-Conventions]] | Unreal folder layout, UCLASS prefixes, Blueprint split |
| [[Toolchain-Research-2026-09-14]] | Unreal + Xcode + Metal toolchain |
| [[Unreal-Networking-Research-2026-09-15]] | Unreal's packet, bunch and channel layer |
| [[Step-1-Move-and-Attack]] | Unreal gameplay framework tutorial |

**What gets harder than Unreal:**

- **No editor.** Levels, spawn points and tuning are files and code; there is no viewport to drag things in.
- **Assets.** Meshes, animation and materials are imported and wired by hand rather than through an asset browser.
- **Breaking changes.** Every upgrade needs a migration pass (§2); staying on one version is possible but freezes bug fixes.
- **Fewer ready-made systems.** Character movement, animation state machines, UI widgets and navigation are crates or
  hand-written, not engine defaults.

**What gets easier:**

- **ECS fits the "weak at OOP structuring" risk** recorded in [[Decisions]]: there is no inheritance tree to design badly.
- **Rust's compiler catches whole classes of mistakes** (ownership, null, data races) that C++ leaves to the developer.
- **One language, one build tool** (`cargo`), no separate editor process, fast iteration for code-only changes.
- **Everything is code**, so the whole game is diffable and reviewable in git — no binary assets for logic.

## 5. NOT verified

| claim | why | what would settle it |
|---|---|---|
| Bevy runs well on this Mac (Apple silicon, macOS 26) | not run yet | `cargo run` a Bevy example on this machine |
| How long a Rust + Bevy learning curve is for a C/C++ developer | opinion, not fact | the first milestone's measured hours |
| Whether Bevy handles a pack of 10+ enemies with effects at a stable frame rate on this Mac | no measurement | a greybox prototype with the real enemy count |
| Whether `vleue_navigator` covers click-to-move well enough (dynamic obstacles, agent radius) | only its description read | build the click-to-move prototype |
| Animation pipeline maturity (glTF import, retargeting, state machines) | not researched | a research pass before the vertical slice |
| Whether `bevy_replicon` covers the co-op model in ADR-007 (join, prediction, interest) | only its description read | a research pass before multiplayer work |

## Sources

- [Bevy News](https://bevy.org/news/) · [Bevy 0.19 release post](https://bevy.org/news/bevy-0-19/) · [Migration Guides introduction](https://bevy.org/learn/migration-guides/introduction/)
- docs.rs: [bevy_replicon](https://docs.rs/crate/bevy_replicon/latest) · [avian3d](https://docs.rs/crate/avian3d/latest) · [vleue_navigator](https://docs.rs/crate/vleue_navigator/latest)

## Related

- [[Decisions]] · [[Design-Doc]] · [[00-Research-Hub]]
