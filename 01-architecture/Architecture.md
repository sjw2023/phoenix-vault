---
type: architecture
project: Phoenix
updated: 2026-08-03
engine: Unreal Engine 5 (C++)
tags: [project/phoenix, gamedev, architecture]
---

# Phoenix — Architecture

> [!warning] **Superseded 2026-09-25 — written for Unreal Engine.** Phoenix moved to Rust + Bevy ([[Decisions]] ADR-008). Kept as history; the reasoning may still be useful, the mechanisms are not. Engine-independent design lives in [[Design-Doc]] and [[Combat]].


The technical layer between [[Design-Doc]] (the *what/why*) and the actual C++ (the *how in detail*), for **Unreal Engine 5**. This is the "where does code go, and how do objects relate?" doc — and it matters more here than it would have in Godot, because Unreal is inheritance-heavy and will happily let you build a tangled class tree. The whole job of this doc is to keep Phoenix on the **composition** side of Unreal. See also [[Coding-Conventions]] and [[System-Design]].

## The one mental model
Unreal games are built from the **Gameplay Framework**: a fixed set of engine classes you subclass and fill in. The pieces you'll actually touch:

- **`AActor`** — anything placeable in a level. The base "thing."
- **`UActorComponent` / `USceneComponent`** — reusable behavior/data you *attach* to an actor (mesh, collision, and your own `UStatsComponent`, `UHealthComponent`, …).
- **`APawn` → `ACharacter`** — an actor that can be possessed and controlled; `ACharacter` adds a capsule + `CharacterMovementComponent`.
- **`AController` → `APlayerController` / `AAIController`** — the "brain" that possesses a pawn. Player input lives on the PlayerController; enemy AI on an AIController.
- **`AGameModeBase`** — the rules/among: which pawn, which controller, win/lose. Server-only.
- **`AGameStateBase` / `APlayerState`** — shared/replicated state.

**The rule that keeps this project sane:** prefer **composition (Actor + Components)** over **inheritance (deep subclass chains)**. Unreal *allows* `AFireSkeleton : ASkeleton : AEnemy : ACharacter`; resist it. When you're about to subclass, ask: *can this be a component instead?* This is the same discipline the Godot plan used — it's just more of a conscious fight in Unreal, because the engine's grain runs toward inheritance.

## Folder structure
Two roots matter: `Source/` (C++, checked into the vault-adjacent repo) and `Content/` (assets + Blueprints, binary, managed in-editor).

```
Phoenix.uproject
Source/
├── Phoenix.Target.cs / PhoenixEditor.Target.cs   # build targets
└── Phoenix/
    ├── Phoenix.Build.cs        # module deps (Core, Engine, EnhancedInput, ...)
    ├── Phoenix.h / .cpp        # module implementation
    ├── Core/                   # GameMode, GameState, PlayerController, GameInstance
    ├── Player/                 # PhoenixCharacter + input
    ├── Enemies/                # enemy actors + AI controllers
    ├── Stats/                  # StatTypes.h, StatsComponent (designed; code reset 2026-09-14, see tag pre-reset)
    ├── Components/             # reusable components: Health, Hitbox, ...
    ├── Items/                  # item + affix data assets, loot rolling
    ├── Skills/                 # skill data + runtime
    └── Passives/               # passive node graph
Content/
├── Maps/                       # levels (.umap)
├── Blueprints/                 # BP_ subclasses of the C++ classes (tuning/wiring)
├── Characters/  Enemies/  UI/
└── Data/                       # DataAsset / DataTable instances (items, affixes, skills)
Config/                         # DefaultEngine.ini, DefaultGame.ini
```

> [!note] Readable copy on the [Phoenix Miro board](https://miro.com/app/board/uXjVHndMfFs=/?moveToWidget=3458764683757480834), drawn as plain boxes: Miro's content filter rejected it twice as a Mermaid diagram (HTTP 403, 2026-09-15), so the board version **shortens exact file names**. **This note is canonical.**

Rule: **C++ defines systems and base classes in `Source/`; Blueprints in `Content/Blueprints/` subclass them for tuning and designer-facing wiring.** New number-affecting content is a **Data asset**, not a new C++ class (see below).

## Composition: Actor + Component
A component is a self-contained unit of data/behavior attached to an actor. An enemy is *assembled*, not inherited:

```
BP_Enemy (subclass of APhoenixCharacter)
├── (inherited) CapsuleComponent, SkeletalMeshComponent, CharacterMovementComponent
├── UStatsComponent      # the numbers spine (designed; code reset 2026-09-14)
├── UHealthComponent     # current/max HP, TakeDamage(), OnDied delegate
├── UHitboxComponent     # deals/receives damage
└── AI: APhoenixAIController possesses it, runs a Behavior Tree
```

> [!note] Readable copy on the [Phoenix Miro board](https://miro.com/app/board/uXjVHndMfFs=/?moveToWidget=3458764683755799281); **this note is canonical.**

A "fast melee" vs a "ranged" enemy are the **same character class + different components/data + a different Behavior Tree** — not two subclasses. `UStatsComponent` and `UHealthComponent` are written once and reused by the player, every enemy, and the boss. That reuse is exactly why the stat spine was built first.

## C++ vs Blueprint split
Both are first-class. The workflow for Phoenix:
- **C++**: systems, base classes, math, anything performance- or correctness-sensitive (the stat spine, loot rolling, damage). Marked `UCLASS`/`UPROPERTY`/`UFUNCTION` so Blueprints can see them.
- **Blueprint subclasses**: create `BP_PhoenixCharacter`, `BP_Enemy`, `BP_PhoenixGameMode` in the editor to set meshes, tune exposed numbers, and wire visuals/animation without recompiling C++.

Expose tunables from C++ with `UPROPERTY(EditAnywhere, BlueprintReadWrite)` so they appear in the Blueprint/Details panel — the Unreal equivalent of Godot's `@export`.

## Global systems: Subsystems (the Unreal-native singletons)
Instead of Godot autoloads, use **Subsystems** — engine-managed singletons with automatic lifecycle:
- **`UGameInstanceSubsystem`** — lives for the whole game session. Home for the **event bus** (a class that only declares delegates like `OnEnemyDied`, `OnItemPickedUp`) and **registries** (loaded item/affix/skill data by id).
- **`UWorldSubsystem`** — per-level. Good for spawn management, the current run's world state.

Access is `GetGameInstance()->GetSubsystem<UPhoenixEventBus>()`. Systems broadcast/subscribe to delegates on the bus rather than holding direct references to each other — the loot system listens for `OnEnemyDied` and never needs to know combat exists.

## Decoupling: delegates
Unreal's signal mechanism is **delegates**:
- `DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnStatsChanged)` — Blueprint-assignable, used by `UStatsComponent::OnStatsChanged`.
- Multicast delegates = many listeners, like Godot signals. Bind in `BeginPlay`.
Prefer a delegate (often on the event-bus subsystem) over a raw pointer to another system.

## The stat system is the spine
> [!warning] **Designed, not currently built.** Implementation reset 2026-09-14 — reference copy at git tag `pre-reset`. Rebuild at step 3.

`Stats/StatsComponent.*` + `Stats/StatTypes.h`. Contract the rest of the game obeys:
- Every entity with numbers owns a **`UStatsComponent`**.
- Base values: `SetBase("life", 50.f)`.
- **Anything** that changes numbers — equip, allocate passive, apply buff — calls `AddModifier(s)` with a `Source` (an `FName` id).
- Removing that source (unequip/respec/expire): `RemoveModifiersFromSource(Source)`.
- Read via `GetValue("stat")`; listen to `OnStatsChanged`.
- Math: `final = (base + Σ FLAT) * (1 + Σ INCREASED) * Π(1 + each MORE)`.

> **Engine-native alternative — GAS.** Unreal ships the **Gameplay Ability System**, whose *Attribute Sets* + *Gameplay Effects* are literally a modifier-aggregation stat system — the idiomatic Unreal answer to exactly this. We are **deliberately not** using GAS for v1: it is powerful but has a famously steep learning curve, and our own `UStatsComponent` is small, understood, and already verified. **GAS is the planned upgrade path** once the game exists and its complexity is justified (skills/abilities especially). Keep the modifier concepts (FLAT/INCREASED/MORE ≈ Additive/Multiplicative GameplayEffect modifiers) so the eventual port is mechanical.

## How a change flows (worked example)
Player equips a sword granting "+12 physical damage, 20% increased physical damage":
1. Equipment component calls `Stats->AddModifiers({...})` with `Source = "Sword_A"`.
2. `UStatsComponent` marks dirty and broadcasts `OnStatsChanged`.
3. The HUD (bound to `OnStatsChanged`) refreshes via `GetValue("physical_damage")`.
4. Next attack, the skill code reads `GetValue("physical_damage")` — already current.
5. Unequip → `RemoveModifiersFromSource("Sword_A")` → numbers revert automatically.

No system needed another system's internals. Components + event-bus subsystem + the stat spine = the whole architecture.

## Decision guide (quick reference)
- New content variant (item/enemy/skill)? → **Data asset** (`UPrimaryDataAsset`/`UDataTable` row), not a C++ subclass.
- New behavior on an actor? → **`UActorComponent`**, not a subclass.
- Two systems need to talk? → **delegate on the event-bus subsystem**, not a direct pointer.
- Something needs global access? → a **Subsystem**, not a global/static.
- New numbers on an entity? → **modifiers on its `UStatsComponent`**, never a raw `float` elsewhere.
- Tempted to write `AFoo : ABar : ABaz`? → stop; make it components + data.
