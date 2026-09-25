---
type: decision-log
project: Phoenix
updated: 2026-09-14
engine: Unreal Engine 5 (C++)
tags: [project/phoenix, gamedev, decisions]
---

# Phoenix — Decisions

A running log of decisions for the Phoenix project. Newest first. See [[HOME]] for the index and [[Design-Doc]] for full rationale.

## 2026-09-25 — Engine switched: Unreal Engine 5 → Rust + Bevy (SUPERSEDES the 2026-08-03 engine decision)

### ADR-008: Build Phoenix in Rust with Bevy
**Status:** Accepted — user, 2026-09-25.
**Context:** Unreal was chosen on 2026-08-03 to play to C++ strength, with the tradeoff recorded that Unreal is
inheritance-heavy and the developer is "weak at OOP structuring". Facts on Bevy: [[Bevy-Research-2026-09-25]].
**Decision:** Phoenix is built in **Rust** with **Bevy 0.19**, pinned through the first milestone.
**Why:**
- **ECS removes the thing Unreal made harder.** Data in components, behaviour in systems; no class hierarchy to
  design badly — the exact risk ADR of 2026-08-03 flagged.
- **Rust's compiler catches ownership, null and data-race mistakes** that C++ leaves to the developer.
- **Everything is code and data files**, diffable in git, editable in vim; the developer wants to wire systems by
  hand as the way to learn them.
- The ecosystem covers the game's needs on 0.19: `avian3d` (physics), `vleue_navigator` (navmesh, click-to-move),
  `bevy_replicon` (server-authoritative networking, matching ADR-007).
**Tradeoffs accepted:**
- **No editor** — levels, spawns and tuning are files, not a viewport ("the upcoming Bevy Editor", 0.19 post).
- **Breaking changes every release** — bevy.org: *"still in the 'experimentation phase'"*. Mitigation: pin 0.19 and
  migrate deliberately.
- Assets, animation, character movement and UI are hand-wired or crates, not engine defaults.
- The Unreal-specific specs are retired (below). Roughly 2,000 lines of design work, kept as history.
**Consequences:**
- **Superseded, kept as history:** [[Combat-Tech]], [[Network-Protocol]], [[OOP-Foundations]], [[Architecture]],
  [[Coding-Conventions]], [[Toolchain-Research-2026-09-14]], [[Unreal-Networking-Research-2026-09-15]],
  [[Step-1-Move-and-Attack]].
- **Unchanged:** [[Design-Doc]], [[Combat]] (rules, damage formula, feedback, edge cases), ADR-007 co-op,
  [[Production-Plan-2026-09-14]] milestones, the documentation framework.
- New notes needed as we go: Rust/Bevy architecture, ECS conventions, and the combat technical spec rewritten for ECS.
**Alternatives considered:** stay on Unreal (rejected — the inheritance grain and the editor-first workflow are not
what the developer wants to learn); Rust without Bevy, engine from scratch (that remains the separate project of
ADR-005).

## 2026-09-15 — Friends can join: co-op multiplayer is in v1

### ADR-007: v1 lets a friend join and play together
**Status:** Accepted — user, 2026-09-15: *"Yes, friend can join"*.
**Context:** Earlier the same day the user chose "option B": server-authoritative architecture that also works as
single-player ([[OOP-Foundations#7. Multiplayer-ready objects (decision: option B)|OOP-Foundations §7]]). Whether
multiplayer actually **ships** in v1 was left open ([[Combat#14. Open questions|Combat §14]] Q1).
**Decision:** v1 supports co-op — a friend can join a game and play together. The architecture stays
server-authoritative, as designed in [[OOP-Foundations]] and [[Network-Protocol]].
**Consequences:**
- The three-mode multiplayer matrix ([[Combat-Tech#11.3 Multiplayer matrix|Combat-Tech §11.3]]) becomes a **v1 release gate**,
  not a readiness check — for every feature.
- A new feature spec is needed: **Sessions and joining** — how a friend finds and joins a game.
- Loot and XP **sharing** must be designed in the Loot and Progression specs.
- Player death needs a co-op rule (revive by a friend or respawn) — open.
- [[Design-Doc]] §3 no longer lists multiplayer as out of scope.
- Networking scope grows: every feature spec now carries a real multiplayer cost.
**Open:** hosting model (one player hosts, or a separate server); maximum players; how friends connect.
**Alternatives considered:** v1 single-player with multiplayer-ready code (the earlier default — replaced by this
decision).

## 2026-09-14 — Toolchain: Unreal Engine 5.8 + Xcode 26.1.1, macOS only

### ADR-006: Use UE 5.8 with the existing Xcode 26.6; target macOS only for now
**Status:** Accepted — **amended 2026-09-14 (r2)**, see Amendment below.

> [!note] **Amendment r2 (2026-09-14).** Tested before downloading Xcode 26.1.1: UE 5.8.2 with the
> **existing Xcode 26.6** gives SDK `Status=Valid`, compiles C++ (`Result: Succeeded`), and starts the editor
> with 0 errors once the Metal toolchain component is installed ([[Toolchain-Research-2026-09-14]] §6a–6c).
> **Xcode 26.1.1 is not installed.** Trade-off accepted: Epic's page says "Xcode 26.4 is not compatible", and
> 26.6 is newer than that, but the engine's own `Apple_SDK.json` accepts up to 27.9.0.
> **Fallback:** if a problem appears at runtime and is traced to Xcode, install 26.1.1 beside 26.6 and select it.
> Original r1 text follows unchanged.
**Context:** Tested on this machine: UE 5.5.4 with Xcode 26.6 cannot compile C++ at all
(`Platform Mac is not a valid platform to build.`, exit 6). No Xcode version is both inside UE 5.5's
accepted range (15.2–16.9.0) and supported by Apple on macOS 26.6.2. Full evidence:
[[Toolchain-Research-2026-09-14]].
**Decision:** Move to **Unreal Engine 5.8** with **Xcode 26.1.1** — the only combination documented
as supported by both Epic (26.1.1 recommended) and Apple (26.1.1 supports macOS Tahoe 26.x).
**Target platform: macOS only.** Windows is out of scope until decided otherwise.
**Consequences:** Large engine download. Xcode 26.1.1 sits beside the existing 26.6. `EngineAssociation`
in the hand-written `.uproject` will be `5.8`. UE 5.5.4 remains installed until 5.8 is verified; removing
it is a separate decision. UE 5.8 is Epic's last planned major UE5 release before UE6.
**Alternatives considered:** UE 5.5 + Xcode 16.4 (Apple does not support 16.4 on macOS 26.6.2);
UE 5.6/5.7 (maximum Xcode unknown, forum reports of build failures); UE 5.8 with the existing Xcode 26.6
(Epic flags 26.4 as incompatible, 26.6 not stated); editing UE 5.5's `MaxVersion` (unsupported, and the
engine compiles with `-Werror`, so a newer compiler's new warnings become errors) — not tested, declined.

## 2026-09-14 — Stay on Unreal; own engine becomes a separate project

### ADR-005: Build Phoenix on Unreal Engine; a custom engine is a separate, later project
**Status:** Accepted.
**Context:** Diablo IV and Path of Exile 2 both run on in-house engines, which raised the question of
whether Phoenix should too. Checked history: Grinding Gear Games was founded in 2006 by **two people**
in a garage; Path of Exile development began ~2007, the studio had 18 staff by 2012, and 1.0 shipped
October 2013 — about six years, full-time, as a business. Same genre on licensed engines: Titan Quest II
(Unreal Engine 5), Last Epoch (Unity). So a small team *can* build its own engine; the cost is years
before the game is playable.
**Decision:** Phoenix stays on **Unreal Engine**. Its purpose is learning *how to build a game* — the
three pillars: combat feel, loot tables, build systems. Writing an engine is a different project that
teaches *how engines work*, and is deferred to a **separate project**, started after Phoenix has
taught the game-building side.
**Consequences:** Unreal's recorded costs stand (inheritance-heavy framework; macOS friction — no Live
Coding, Xcode version range). Engine curiosity goes into the future project, not into Phoenix.
**Alternatives considered:** custom engine for Phoenix (rejected — years of infrastructure before any
pillar is testable); a thin C framework such as SDL or raylib (rejected *for Phoenix*, noted as a good
starting point for the future engine project); Unity (not re-evaluated — Unreal already fits the C++
strength and 3D top-down view).
**Sources:** [Grinding Gear Games](https://en.wikipedia.org/wiki/Grinding_Gear_Games) ·
[Path of Exile](https://en.wikipedia.org/wiki/Path_of_Exile) ·
[Path of Exile 2](https://en.wikipedia.org/wiki/Path_of_Exile_2) ·
[Diablo IV](https://en.wikipedia.org/wiki/Diablo_IV) ·
[Titan Quest II on UE5](https://www.neowin.net/news/thq-nordic-unveils-titan-quest-ii-a-sequel-being-built-on-unreal-engine-5/) ·
[Last Epoch](https://en.wikipedia.org/wiki/Last_Epoch) — checked 2026-09-14.

## 2026-09-14 — Git + LFS from commit #1; project reset to empty

### ADR-002: Use Git + Git LFS for version control
**Status:** Accepted.
**Context:** Git stores compressed binary files as full copies rather than deltas — measured on this
machine: a file 6.5× *smaller* on disk produced a repo 6.7× *larger* over 10 commits. Unreal's
`.uasset`/`.umap` are exactly that shape. Git history is append-only, so the cost is permanent and
paid by every clone. Full mechanism and evidence: [[Git-LFS-Research-2026-09-14]].
**Decision:** git + Git LFS, with `.gitattributes` LFS rules in place **before the first commit** —
LFS only intercepts files matching a rule at `git add` time, so a late rule means a permanent full
blob and a history rewrite.
**Consequences:** git-lfs must be installed on any machine that clones (otherwise checkouts yield
pointer files). Hosted LFS quota applies once a remote exists — unquantified, see research §7.
**Alternatives considered:** plain git (repo bloat, rejected); **Perforce** (industry standard for
game studios, natively integrated in Unreal — rejected: needs a server, solves team-scale problems
this project does not have).

### ADR-003: Reset the project to empty and rebuild by hand
**Status:** Accepted.
**Context:** The scaffold and the ported POE stat spine were generated rather than written. This is
a learning project; the point is understanding every file, not possessing it.
**Decision:** Delete `Source/`, `Config/`, `Phoenix.uproject`; rebuild from nothing, by hand.
**Consequences:** The verified stat spine (24.48 / 80.0 / 6.0) leaves the working tree. It is
**not lost** — commit `8f929e3`, tag `pre-reset`, recoverable per-file with
`git checkout pre-reset -- <path>`. `_godot_archive/` deliberately kept, per the 2026-08-03 decision
that it is archived rather than deleted.
**Alternatives considered:** keep the scaffold and learn by modifying it — rejected, because reading
generated code is a weaker teacher than writing it.

### ADR-004: Adopt Diátaxis as the documentation framework
**Status:** Accepted.
**Context:** No stated frame for what goes in a note; the vault was drifting toward one flat pile.
**Decision:** Adopt **Diátaxis** (<https://diataxis.fr>) — Tutorial / How-to guide / Technical
reference / Explanation — plus ADRs for decisions, which Diátaxis does not cover. Vault restructured
into numbered clusters matching the RPS / Vault-API / VCM convention. See
[[Documentation-Framework]].
**Consequences:** Notes name their form; mixing two forms in one note is now a defect. All existing
notes renamed and relocated; 39 wikilinks rewritten and verified.
**Alternatives considered:** keep the flat `Phoenix XXX.md` layout (rejected — a learning project
generates research notes steadily, and flat does not survive that); invent a bespoke structure
(rejected — three sibling vaults already share a convention).

## 2026-08-03 — Engine switched: Godot → Unreal Engine 5 (SUPERSEDES the 2026-07-13 engine decision)

**Decision:** Build Phoenix in **Unreal Engine 5** with **C++**, as a **3D top-down** ARPG (like the real Path of Exile), not 2D.

**Why the change:**
- **Plays to the strongest language.** Dev is strongest in C/C++; Unreal is a C++-first engine, so the primary systems language is now the one the dev is best at (vs. learning GDScript).
- **3D top-down matches the POE reference** directly, and 3D is Unreal's native strength (rendering, lighting, asset pipeline).
- **Industry-standard engine** with deep C++ gameplay framework and tooling.

**Tradeoff explicitly accepted:**
- The original Godot choice was partly to shield the **"weak at OOP structuring"** spot via Godot's composition-first node model. **Unreal leans the opposite way** — it is built on a deep class inheritance hierarchy (`UObject → AActor → APawn → ACharacter`, `UActorComponent`, `UGameMode`, etc.), and idiomatic Unreal means working *with* that class tree. This engine asks **more** OOP structuring, not less. Mitigations: lean on Unreal's **component** model (Actor + Components) wherever possible instead of subclassing, and keep the [[Coding-Conventions]] + [[Architecture]] docs as the guardrails. This is the main risk to watch on this project.
- Heavier engine, larger installs/build times, steeper first-run setup (needs a C++ toolchain — Visual Studio on Windows / Xcode on macOS).

**Consequences:**
- The stat-aggregation spine was **ported from GDScript to C++** (`UStatsComponent` + `FStatModifier`), same POE math, same verified values (24.48 / 80.0 / 6.0).
- Technical docs rewritten for Unreal. The Godot project and `.gd` code are **retired/archived**, not deleted.

## 2026-07-13 — Original decisions (engine choice SUPERSEDED above; rest still stand)

### Project
First game — a **POE2-like ARPG**. v1 = learning-focused **vertical slice**, not full POE2. *(Still stands.)*

### Engine — Godot 4 + GDScript *(SUPERSEDED 2026-08-03 → Unreal 5)*
Originally chosen because the node/scene model imposes structure (dev weak at OOP structuring), free rendering/UI/physics + 1-click PC export, fast iteration. Alternatives libGDX (Java) and Rust+Bevy were declined. This reasoning was overridden by the decision to build in C++ (the dev's strongest language) and to go 3D top-down.

### Dev background
Strongest in C/C++, primary language Java, weak at OOP structuring, first game. *(Still stands — and is the reason C++ now appeals.)*

### Architecture
Data-driven design; composition over inheritance; the **stat-aggregation system is the spine**. *(Still stands — now expressed with Unreal Actors + Components and DataAssets; see [[Architecture]].)*

### Memory home
**Obsidian vault is the single source of truth** for Phoenix. Notion was a one-time transport only. *(Still stands.)*
