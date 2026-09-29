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

- **Checkpoint 10 — five enemies.** A position table and a `for` loop in `enemy_spawn`. **No other file
  changed** — chase, both attack systems and death already iterate, so five enemies cost zero new behaviour
  code. The clearest payoff so far from components-plus-systems over an inheritance hierarchy.
- **Handles, not assets.** `meshes.add(...)` moved above the loop and each enemy gets `mesh.clone()`.
  Cloning a `Handle<Mesh>` is a refcount bump; calling `add` inside the loop would upload five identical
  cubes. Same failure family as the `world_spawn` DeviceLost — assets created more often than intended.
- **Two decisions got made by accident, not by design:** `player_attack` loops over every enemy in range,
  so cleave is now the default; and five enemies at 8 damage / 1.2 s kills a 100 HP player in ~3 s, so
  player death stopped being deferrable. An unmade decision does not stay unmade — it gets made by
  whoever wrote the loop.

- **Checkpoint 11 — state machine.** `AppState { Playing, GameOver }`. Gameplay systems carry
  `.run_if(in_state(AppState::Playing))`; player and enemy spawns moved from `Startup` to
  `OnEnter(AppState::Playing)`, so **restart is a state transition** and there is only ever one code path
  that creates an actor. `OnExit(AppState::Playing)` despawns everything with `Player` or `Enemy`.
  `world` and `camera` stay on `Startup` — they belong to the app, not to a run.
- **Bugs that compiled cleanly:** `run_if(AppState::Playing)` (a run condition is a function call —
  `in_state(..)`); `Or<With<A>>, With<B>>` (`Or` takes one tuple, and `Query` only ever has two type
  parameters); `enemy_chase` left ungated so enemies kept chasing the corpse; and
  `next_state.set(AppState::Playing)` inside `death` — a no-op transition that would have looked like
  "states don't work." The type checker verified all four.
- **Checkpoint 12 — single-target attack.** `player_attack` was damaging every enemy in range from one
  timer tick: four `hit enemy` lines in the same millisecond. Player dps was `31.25 x N`, enemy dps
  `6.67 x N` — a **fixed 4.7:1 ratio regardless of N**, so enemy count had no effect on difficulty at all.
  Replaced with nearest-target selection: an immutable scan recording `(distance, Entity)`, then one
  `enemies.get_mut(target)` write. Read-then-write is the ECS shape for "pick one of many and act on it".
- **Measured after the fix:** five enemies take the player 100 to -12 HP in ~5 s, `You Died` fires, `R`
  restarts with a fresh player and five fresh enemies. Enemy count now means something.
- **Design settled:** the basic attack is single-target; cleave becomes an *ability* that pays for hitting
  several targets. This is what the parked "both" answer actually meant.

- **Checkpoint 13 — death modes.** `DeathMode { Softcore, Retry, Hardcore }` as a **resource**, not a state
  — it is a value systems read, not something with `OnEnter`/`OnExit` transitions. `init_resource` registers
  it; `init_state` is the other one. Keys 1/2/3 switch it, including from the game-over screen.
  All three verified working.
- **`Health` became `{ current, max }`.** A tuple struct is right for one field whose meaning the type name
  already carries (`Speed(f32)`); wrong the moment there are two, because `.1` tells a reader nothing.
  Softcore respawn is `health.current = health.max` — the component knows what full means, so `100.0`
  is not written down in a second place.
- **Respawn also clears `MoveTarget`.** State-as-component-presence means respawning has to remove the
  state components too, or you teleport home and immediately walk back to where you died.
- **`match *mode` is exhaustive** — adding the parked ally-revive variant later will make the compiler point
  at this block and refuse to build until it is handled. Open-Closed with a safety net.

### Recorded debt
- `death` both detects death and applies the player's respawn policy, so `combat` knows where the player
  spawns. The clean split is `death` announcing that the player died and `player` deciding what that means
  — Bevy's message system. Deferred deliberately: learning messages while debugging 12 errors was the
  wrong order.

- **Checkpoint 14 — health bar.** Bevy UI is entities and components like everything else: a dark `Node`
  sized in `px`, with a red child `Node` sized in `percent(100)` of it, and a system that writes
  `fill.width = percent(fraction * 100.0)`. No 2D camera needed — a root node without `UiTargetCamera`
  renders to *"the highest order camera targeting the primary window"*, which is the `Camera3d`.
  `clamp(0.0, 1.0)` matters because health goes negative before death is processed.

### Toolchain lesson — a missing `mod` looks like a broken LSP
- Neovim completion in `ui.rs` offered only buffer words (`abc` icon). rust-analyzer was healthy the whole
  time: attached, idle at 0% CPU with a ~1.8 GB index, both proc-macro servers up, hover working in
  `main.rs`. The cause was **no `mod ui;` in `main.rs`** — the file was not part of the crate, so the
  server had nothing to say about it.
- Third occurrence of the same root cause in one day, each wearing a different costume: a compiler error
  (`camera.rs`), a suspicion (`App.rs`), a dead completion menu (`ui.rs`).
- **Habit:** write the `mod` line the moment the file is created, before writing anything in it.
- **Tell:** the editor acting dumb in one file while working in others is a missing `mod`, never the LSP.
- Diagnostic worth keeping: `:lua =vim.tbl_map(function(c) return c.name end, vim.lsp.get_clients({bufnr=0}))`
  prints just client names — dumping the whole client object only shows the first one.

- **Checkpoint 15 — messages, and the debt paid.** `#[derive(Message)] pub struct PlayerDied;` declared in
  `combat.rs` beside its writer, registered once with `.add_message::<combat::PlayerDied>()`.
  `death` now only detects death and writes the message — it lost `DeathMode`, `NextState`, `PLAYER_SPAWN`,
  `MoveTarget`, `&mut Health` and `&mut Transform`, six dependencies, and its query went back to immutable.
  `player.rs` owns the respawn policy; `ui.rs` empties the bar. **Verified: `combat.rs` no longer mentions
  `PLAYER_SPAWN`, `DeathMode` or `NextState` at all.** The behaviour barely changed — the point was that
  `combat` stopped knowing things it had no business knowing.
- **Each `MessageReader` tracks its own read position**, so two readers both see every message. Adding a
  third reaction later is a new reader and zero edits elsewhere.
- **Handlers must be idempotent** unless ordered. `death` can write `PlayerDied` several times before the
  handler restores health, because writer and reader order is not guaranteed across plugins. Setting health
  to max twice equals once, so it is safe here; a handler that *spends* something would need `.chain()`,
  system sets, or a marker component.
- **Wiring is a second declaration, and the failure is quiet.** A file needs `mod`, a system needs
  `add_systems`, a plugin needs `add_plugins`, a message needs `add_message`, a resource needs
  `init_resource`. `clear_health_bar`, then `spawn_game_over`/`despawn_game_over`, were each written and
  left unregistered. In a Bevy project a `never used` warning on a function you just wrote almost always
  means a missing registration.

### Next up
- [ ] Game-over screen (text UI) — written, needs registering.
- [ ] Enemy separation — they currently pile into one cube.

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
