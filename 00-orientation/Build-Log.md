---
type: build-log
project: Phoenix
updated: 2026-09-26
engine: Unreal Engine 5 (C++)
tags: [project/phoenix, gamedev, buildlog]
---

# Phoenix — Build Log

A running, dated log of what actually got built. Newest entries at the top. See [[HOME]] for the index.

## 2026-09-26 — First Bevy code running: walking skeleton and a moving cube
- **Rust toolchain:** installed via Homebrew (rustc + cargo 1.98.1, no rustup — so no per-project toolchain pinning).
- **Project created:** `cargo init --name phoenix` + `cargo add bevy` → Bevy **0.19.1**, edition 2024, with Bevy's
  recommended dev profile (`opt-level = 1`, dependencies at 3). `.gitignore` rewritten for Rust (`/target`,
  `Cargo.lock` kept tracked).
- **Checkpoint 1 — walking skeleton:** window opens; ground plane, point light, camera and a cube render.
  Confirms Bevy runs on this Mac (the last NOT-verified item in [[Bevy-Research-2026-09-25]] §5).
- **Checkpoint 2 — movement:** `Player` marker component, an `Update` system with
  `Query<&mut Transform, With<Player>>` and `time.delta_secs()`; the cube slides frame-rate independently.
- **Errors hit while learning, and what they taught:**
  - `(Cuboid, Color, Transform)` is not a `Bundle` — shapes and colours are **data**, not components; they become
    assets (`meshes.add`, `materials.add`) wrapped in `Mesh3d` / `MeshMaterial3d`.
  - Nothing rendered at first: the `setup` system existed but was never registered with `.add_systems(Startup, setup)`.
- **Learned so far:** App and plugins, Startup vs Update schedules, entities as component tuples, marker components,
  queries with filters, resources (`Res<Time>`), and why `derive` is what makes a struct a Component.

### Next up
- [ ] Click-to-move: cursor → world ray → ground point, then move the cube toward it.

## 2026-09-14 — Version control, vault restructure, project reset to empty
- **Toolchain verified (not installed — already present):** UE 5.5 at `/Users/Shared/Epic Games/UE_5.5` (59 GB), Xcode 26.6, macOS 26.6.2. The old "install UE5 + toolchain" task was stale.
- **Risk found:** UE 5.5 declares `MaxVersion 16.9.0` for Xcode (`Engine/Config/Apple/Apple_SDK.json`); installed Xcode is **26.6**, outside the supported range. Unresolved — will surface on first compile. Mitigations: install Xcode 16.2 alongside, or move to a newer UE.
- **Git + LFS set up** before the editor was ever opened, so `.gitattributes` rules exist from commit #1 — the ordering that makes LFS work at all. git-lfs 3.8.0. See [[Git-LFS-Research-2026-09-14]].
- **Project reset to empty.** `Source/`, `Config/`, `Phoenix.uproject` removed; rebuilding by hand as a learning exercise. Everything recoverable at tag `pre-reset` (`git checkout pre-reset -- <path>`). `_godot_archive/` kept per [[Decisions]].
- **Vault restructured** from flat `Phoenix XXX.md` files into numbered clusters (`00-orientation`, `01-architecture`, `02-build-steps`, `03-research`) with `HOME.md`, matching the RPS / Vault-API / VCM convention. All 39 wikilinks rewritten and verified to resolve.
- **[[Documentation-Framework]] written** — Diátaxis (Tutorial / How-to / Reference / Explanation) plus ADRs, with templates. The frame for every note from here on.

### Next up
- [ ] Hand-write `Phoenix.uproject`, `Phoenix.Target.cs`, `PhoenixEditor.Target.cs`, `Phoenix.Build.cs` — understanding every field rather than pasting.
- [ ] First compile → find out whether Xcode 26.6 is actually a blocker.
- [ ] Build step 1 against [[Step-1-Move-and-Attack]].

## 2026-08-03 — Engine switch: Godot → Unreal Engine 5 (C++, 3D top-down)
- **Decision reversed:** moved off Godot to **Unreal Engine 5 / C++ / 3D top-down**. Rationale + accepted tradeoffs recorded in [[Decisions]] (headline: plays to C++ strength and matches the POE reference; cost is Unreal's inheritance-heavy model vs. the OOP-structuring weak spot).
- **Stat spine ported to C++:** `Source/Phoenix/Stats/` — `EModifierType`, `FStatModifier`, `UStatsComponent` (attachable component), plus an automation test `Phoenix.Stats.Aggregation`. Same POE math; re-verified the values standalone: physical_damage 24.48, life 80.0, 6.0 after unequip.
- **UE5 C++ project scaffolded:** `Phoenix.uproject`, build targets, `Phoenix` module, `Config/`, and a skeleton `APhoenixGameMode`. Opens + compiles in Unreal.
- **Technical docs rewritten for Unreal:** [[Architecture]], [[Coding-Conventions]], [[System-Design]], [[Step-1-Move-and-Attack]].
- **Godot project retired** (archived on disk, not deleted).

### Next up
- [ ] Install UE5 + C++ toolchain; open `Phoenix.uproject` from `/Users/joowon/Workspace/phoenix` and let it build.
- [ ] Run the `Phoenix.Stats.Aggregation` automation test → expect 3 passes.
- [ ] Build step 1 (move & attack) against its spec.

## 2026-07-14 — Technical documentation written (pre-code) [Godot-era, since rewritten]
- Prepared the first technical doc layer (Architecture, Coding Conventions, System Design, Step 1) — originally Godot-specific; rewritten for Unreal on 2026-08-03.

## 2026-07-13 — Vault set up + stat system in place [Godot-era]
- Copied the handoff from Notion into the Obsidian vault; Obsidian is the sole source of truth.
- Created the core notes: [[HOME]], [[Design-Doc]], [[Decisions]], and this build log.
- Stat-aggregation system first coded in GDScript (`scripts/stats/`); later ported to C++ (see 2026-08-03).
