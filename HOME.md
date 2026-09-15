---
type: project
status: active
genre: ARPG (POE2-like)
engine: Unreal Engine 5 (C++)
view: 3D top-down
platform: PC / Desktop
created: 2026-07-12
updated: 2026-09-14
tags: [project/phoenix, gamedev]
---

# 🔥 Phoenix

> [!info] My first game project — a **POE2-like action RPG**, **3D top-down**. Learning-focused. Build a **vertical slice**, not the full POE2.

## Map of content

### Orientation — `00-orientation/`
- [[Design-Doc]] — full GDD (vision, systems, scope, build order)
- [[Decisions]] — decision log / ADRs (engine switch Godot→Unreal, genre, memory home)
- [[Build-Log]] — running notes as things get built
- [[Documentation-Framework]] — **how notes in this vault are shaped** (Diátaxis + templates)

### Architecture — `01-architecture/`
- [[Architecture]] — UE5 gameplay framework, Actor+Component pattern, subsystems, the stat spine
- [[Coding-Conventions]] — Unreal C++ style guide (prefixes, UPROPERTY, memory, delegates)
- [[System-Design]] — per-system data schemas (stats, loot, inventory, skills, passives)
- [[OOP-Foundations]] — **object-oriented design rules for every class**: responsibilities, ownership, layers, multiplayer-ready authority
- [[Combat-Tech]] — combat technical design *(pilot)*
- [[Network-Protocol]] — **every network message across all features**: commands, events, replicated state, rules

### Game design — `05-game-design/`
- [[Combat]] — combat game design *(pilot of the feature-spec template)*

### Build steps — `02-build-steps/`
- [[Step-1-Move-and-Attack]] — technical spec for build step 1

### Claude memory — `99-claude-memory/`
- [[99-claude-memory/README|Claude project memory (copy)]] — snapshot of Claude's working memory for this project; synced 2026-09-15

### Diagrams — Miro
- [Phoenix Miro board](https://miro.com/app/board/uXjVHndMfFs=/) — readable copies of vault diagrams; **the vault stays canonical**

### Production — `04-production/`
- [[Production-Plan-2026-09-14]] — **milestones M0–M5 with exit criteria, risks, research schedule**

### Research — `03-research/`
- [[00-Research-Hub]] — index of every research note
- [[Git-LFS-Research-2026-09-14]] — why git needs LFS for game assets
- [[Toolchain-Research-2026-09-14]] — which Unreal + Xcode can build on this Mac
- [[Game-Production-Research-2026-09-14]] — how studios plan, compared with the design doc
- [[Unreal-Networking-Research-2026-09-15]] — how Unreal turns a gameplay call into UDP packets


## Pillars
1. **Satisfying combat** — responsive move + click-attack, clear hit feedback
2. **Meaningful loot** — frequent, varied, occasionally exciting randomized drops
3. **Build expression** — skills + passives make characters play differently

## Build order (each step playable)
1. Move & attack → 2. Enemies & combat → 3. **Stat system (the spine)** → 4. Loot drops → 5. Inventory & equipment → 6. Skills → 7. Passive tree → 8. Zone & boss (**v1 done**) → 9. Polish

## Now / next

> [!tip] **Current milestone: M0 — Pre-production (paper only).** Exit criteria: [[Production-Plan-2026-09-14#M0 — Pre-production (paper only)|Production Plan M0]].

- [x] Design doc + decisions written
- [x] **Engine switched to Unreal Engine 5 / C++ / 3D top-down** (2026-08-03) — see [[Decisions]]
- [x] ~~UE 5.5 + Xcode installed~~ — tested: UE 5.5.4 + Xcode 26.6 **cannot compile** ([[Toolchain-Research-2026-09-14]])
- [x] Toolchain decided: **UE 5.8 + Xcode 26.6, macOS only** — [[Decisions]] ADR-006
- [x] **Version control set up** — git + LFS, rules in place before the first asset (2026-09-14)
- [x] Vault restructured into numbered clusters, matching the RPS / Vault-API convention (2026-09-14)
- [x] [[Documentation-Framework]] written
- [x] Engine question settled: **stay on Unreal**; own engine = separate later project — [[Decisions]] ADR-005
- [ ] **Project reset to empty (2026-09-14)** — rebuilding from scratch by hand as a learning exercise. Old scaffold recoverable at git tag `pre-reset`.
- [ ] Hand-write `Phoenix.uproject` and the module/target files, understanding each field
- [x] UE 5.8.2 installed; C++ compile verified with the existing Xcode 26.6; Metal toolchain installed; editor reaches Project Browser (2026-09-14)
- [x] ~~Install Xcode 26.1.1~~ — not needed: UE 5.8 works with Xcode 26.6 ([[Decisions]] ADR-006 r2)
- [ ] Compile Epic's blank template with the new toolchain → proves the toolchain before any Phoenix file is written
- [x] Milestone structure adopted — [[Production-Plan-2026-09-14]] (draft, awaiting Comment step)
- [x] [[Design-Doc]] v0.3
- [ ] ~~Build step 1 (move & attack)~~ → now **M1 First Playable** (greybox combat prototype)


## Architecture reminders (fights "weak at OOP structuring" — extra important in Unreal)
- **Composition over inheritance**: Actor + Components (Stats, Health, Hitbox), NOT deep `A...:A...:A...` subclass trees. Unreal's grain runs toward inheritance — consciously resist it.
- **Data-driven**: new items/enemies/skills = Data assets (`UPrimaryDataAsset`/`UDataTable`), not new C++ classes
- **Stat aggregation is the spine**: items/passives/buffs just *add modifiers* to a `UStatsComponent`
- **GAS is the later upgrade path** for stats/skills; we use a simple `UStatsComponent` for v1
