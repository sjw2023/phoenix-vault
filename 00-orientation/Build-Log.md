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

- **Checkpoint 3 — click-to-move works.** Cursor → `viewport_to_world` ray → `plane_intersection_point` on the
  ground plane → `MoveTarget(Vec3)` inserted on the player → a movement system walks there and removes the
  component on arrival. Clicking mid-walk retargets, because `insert` replaces.
- **Design that fell out of ECS:** state is the presence of a component. No `is_moving` flag — having `MoveTarget`
  *is* walking, and the query skips entities without it. Re-usable by enemies unchanged.
- **Bug found:** `move_player` was registered twice (`add_systems` twice), so it ran twice per frame at double
  speed. Bevy does not warn about duplicate registration.

- **Checkpoint 4 — follow camera.** Fixed offset above/behind the player, `look_at` each frame. Hit Bevy's
  documented **B0001** first: `&Transform` and `&mut Transform` in one system is refused at startup because
  Bevy cannot prove the two queries are disjoint. Fixed with `Without<Player>` on the camera query.
- **Checkpoint 5 — facing.** `look_to(to_target, Vec3::Y)` before the translation step, so direction is never
  zero. Bevy's forward is **-Z**; a child cuboid at `z = -0.7` makes the rotation visible.
- **Checkpoint 6 — a second entity.** An enemy with the *same* `MoveTarget` component walks with zero new
  movement code — `move_player` was renamed `move_to_target` because the name had been a lie since it was
  written. `const SPEED` became a `Speed` component: speed is a property of an entity, not of the program.
- **Checkpoint 7 — combat.** `Health` / `Damage` / `AttackTimer(Timer)`, mirrored attack systems, and a
  `death` system using `Option<&Player>` to branch on who died. `despawn()` takes the entity's `Children`
  with it. Enemy dies in three hits; player death currently only logs.

### Open design decision
- **What happens when the player reaches 0 HP.** Respawn at a checkpoint / friend-revive / hardcore. Ties to
  the co-op death rules parked earlier. Currently logs and does nothing.

- **Checkpoint 8 — plugin split.** `main.rs` went from 235 lines to 20: six feature plugins
  (`world`, `player`, `enemy`, `movement`, `combat`, `camera`), each owning its components, its spawn and
  its systems. `main.rs` now contains only `mod` lines and one `add_plugins` tuple.
- **Rule that settled ownership:** the module that defines the *behaviour* owns the component; everyone else
  imports it. Reached the hard way — `Speed` briefly existed in two modules, and `movement::Speed` and
  `player::Speed` are different types, so the player would have silently stopped moving with no error.
- **Rust visibility, both directions:** a child module sees its parent's private items; a parent cannot see
  the child's. `pub struct Foo(pub f32)` needs both `pub`s — a public struct does not get public fields.
  `mod X;` appears exactly once per file in the program, in the parent that owns it; everywhere else it is `use`.

### Regression found by the refactor check
- `world_spawn` was registered on `Update` instead of `Startup` — a fresh ground plane and **point light every
  frame**. `cargo check` was green throughout. Symptom was not a leak warning but
  `ERROR bevy_render: Caught DeviceLost error` after ~4.5 s, preceded by the light clusterer doubling its
  Z slice list (1024 to 2048) and index list (65536 to 131072). It presented as "it broke after the enemy
  died", which was coincidence — the log timestamps show the crash landed while the enemy still had 35 HP.
- **Lesson:** compiling is not working, and two clocks running at similar rates look like cause and effect.

- **Checkpoint 9 — WASD.** Keyboard movement alongside click-to-move; holding a key removes `MoveTarget` so
  the two schemes do not fight. `Dir3::new(direction)` does three jobs in one line: rejects the zero vector
  (= no key held), normalizes diagonals so W+D is not 1.41x speed, and yields the type `look_to` wants.
  A type that cannot hold an invalid value replaces a check you would otherwise have to remember.
- **W is `-Z`**, because the camera sits at `+Z` looking back at the origin. Correct only while the camera's
  orientation is fixed — camera rotation would force this into camera-relative space.

### Next up
- [ ] Multiple enemies — proves the plugin split, and surfaces cleave + player-death as real decisions.

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
