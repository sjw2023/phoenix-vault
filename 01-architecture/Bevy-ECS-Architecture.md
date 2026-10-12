---
title: Bevy ECS — the architecture Phoenix is actually built on
date: 2026-10-12
status: draft
revision: r1
tags: [phoenix, architecture, bevy, ecs, rust]
engine: Bevy 0.19.1
---

# Bevy ECS — the architecture Phoenix is actually built on

Replaces [[OOP-Foundations]] for the Bevy era. That note described an object graph —
`AActor` subclasses owning their own data and methods. **Bevy has no object graph**, and almost
every confusion while building M1 traced back to still looking for one.

Everything below is illustrated with code already in `~/Workspace/phoenix`.

## The whole model in one picture

```mermaid
flowchart TB
  W["World — the single global store"]
  E["Entity<br/>just an id (index + generation)"]
  C["Component<br/>plain data, keyed by TYPE"]
  R["Resource<br/>exactly one global value"]
  M["Message<br/>a one-shot announcement"]
  S["System<br/>a plain fn; params declare its access"]
  Sc["Schedule<br/>Startup / Update / OnEnter / OnExit"]
  P["Plugin<br/>a registration script, runs once"]

  W --- E
  W --- R
  W --- M
  E --- C
  S -- "reads and writes" --> W
  Sc -- "decides WHEN" --> S
  P -- "registers" --> S
  P -- "registers" --> R
```

## The six concepts, in Phoenix's own code

| concept | what it is | in Phoenix |
|---|---|---|
| **Entity** | an id. 8 bytes, `Copy`. Holds no data and grants no access. | the player, each enemy, each damage number |
| **Component** | plain data attached to an entity, stored **keyed by type** | `Player`, `Enemy`, `Health`, `Speed`, `MoveTarget`, `Knockback`, `HitFlash` |
| **Resource** | exactly one global value of its type | `DeathMode`, `EnemyMaterials`, `HitStop`, `Time`, `ButtonInput<KeyCode>` |
| **Message** | a one-shot announcement; every reader keeps its own cursor | `PlayerDied`, `Hit` |
| **System** | an ordinary `fn`; its **parameters declare what it touches** | `move_to_target`, `enemy_chase`, `player_attack`, `death` |
| **Schedule** | when a system runs | `Startup`, `Update`, `OnEnter(Playing)`, `OnExit(Playing)` |

## There is no object graph

`Player` is the entire definition of a player:

```rust
#[derive(Component)]
pub struct Player;
```

Zero fields. It is a **label**. The player entity is an id with independent components sitting
beside each other — `Transform`, `Speed`, `Health`, `Damage`, `AttackTimer`, `Mesh3d`, `Player` —
and none is inside another.

Consequences that keep coming up:

- **Behaviour attaches to a component set, not to a type.** `move_to_target` queries
  `(Entity, &mut Transform, &MoveTarget, &Speed)` and therefore moves the player *and* the enemies
  with no knowledge of either. Five enemies cost zero new behaviour code.
- **Two identical structs in different modules are different types.** `movement::Speed` and
  `player::Speed` would never match the same query — the near-miss that would have silently
  stopped the player moving.
- **State is the presence of a component.** No `is_moving` flag: having `MoveTarget` *is* walking,
  having `Knockback` *is* being knocked back. `insert` enters the state, `remove` leaves it, and a
  system does one frame of work for whoever is currently in it.

## A plugin owns nothing

`EnemyPlugin::build()` runs **once**, at `add_plugins` time, calls `add_systems` / `init_resource`,
and is then finished permanently. It does not wrap, own or mediate the systems it registered.
Paste its three lines into `main.rs` and the game is identical; the plugin exists so registrations
sit beside the code they register.

Evidence there is no plugin scoping — the same call appears under four different plugins, all
acting on the one World:

```
combat.rs:172   commands.entity(entity).despawn();        CombatPlugin
enemy.rs:164    commands.entity(entity).try_remove::<…>() EnemyPlugin
player.rs:102   commands.entity(entity).remove::<…>();    PlayerPlugin
ui.rs:134       commands.entity(entity).despawn();        UIPlugin
```

## Systems get their arguments by type

`SystemParam` is type-directed injection — Bevy reads the parameter types and supplies them.
`Commands`, `Res<T>`, `ResMut<T>`, `Query<D, F>`, `Single<D, F>`, `MessageReader<T>`,
`MessageWriter<T>` are all `SystemParam`.

Two consequences:

1. **Parallelism is free and automatic.** `&T` versus `&mut T` in the signature is the whole
   declaration; the scheduler runs non-conflicting systems concurrently with no locks.
2. **Conflicts are refused, not risked.** Two params touching the same component where one is
   `&mut` is error **B0001** at startup. Fix by proving the sets disjoint (`Without<Player>` on the
   camera query) or, when the sets are genuinely the same, by a two-pass snapshot
   (`enemy_separation`).

## Commands are a queue, not an action

`commands.entity(id)` performs **no lookup and no validation** — it returns an `EntityCommands`
that remembers the id and a handle to the queue. The operation is applied at the next sync point,
where the entity physically moves between archetypes.

That deferral is why structural changes cannot be immediate: moving an entity between archetypes
mid-iteration would invalidate the query being looped over.

And it is why **a query guard cannot protect a command**: the despawn you are racing is *also*
queued, so "does it exist now" does not answer "will it exist when my command lands". The only
answer is to tell the queued command what to do on arrival — `try_insert`, `try_remove`,
`try_despawn`.

> **Rule:** `insert` / `remove` / `despawn` when you own the entity's lifetime; the `try_` variant
> when you learned the entity from a message, a stored `Entity`, or another system's output.

## Nothing in Phoenix draws anything

No system here renders. They change `Transform.translation`. Bevy's renderer reads each entity's
`GlobalTransform` every frame and draws the mesh wherever that says.

**To move something, change its data — the picture follows.** `Transform` is the editable local
description (`translation: Vec3`, `rotation: Quat`, `scale: Vec3`); `GlobalTransform` is the
composed world-space matrix (`Affine3A`) Bevy derives from the hierarchy.

## Where the clock comes from

`Res<Time>` is **virtual** time — scalable and pausable, and every `delta_secs()` in the project
reads it. `Time<Real>` is wall-clock. Anything measuring how long an effect *on time* should last
must read the clock that effect does not touch — the hitstop timer is the lived case.

## Why Rust works this way — OOP, minus inheritance

Source: *The Rust Programming Language*, ch. 18.1 *Characteristics of Object-Oriented Languages*
and ch. 18.3 *Implementing an Object-Oriented Design Pattern* (checked 2026-10-12).

Rust has two of the three classic OOP characteristics and deliberately drops the third.

| characteristic | Rust | the Book's words |
|---|---|---|
| **Objects** (data + behaviour) | yes | *"Structs and enums have data, and `impl` blocks provide methods on structs and enums."* |
| **Encapsulation** | yes | *"If encapsulation is a required aspect for a language to be considered object oriented, then Rust meets that requirement."* |
| **Inheritance** | **no** | *"There is no way to define a struct that inherits the parent struct's fields and method implementations without using a macro."* |

The two jobs inheritance did are split:

- **Code reuse** → **default trait methods**: a trait ships an implementation every implementor gets
  and may override.
- **Substitutability** → **trait bounds** (the Book calls the generic form *bounded parametric
  polymorphism*) or **trait objects** (`dyn Trait`) at runtime.

`impl Plugin for EnemyPlugin` is that polymorphism: `add_plugins` accepts anything implementing
`Plugin`, and the six plugins are substitutable through it — an interface satisfied by unrelated
types, not a hierarchy under a common parent. Note the direction, `impl Trait for Type`, written
apart from the type: a type can implement a trait declared in a crate it has never heard of.

The Book is blunt about the reason: *"Inheritance is often at risk of sharing more code than
necessary. Subclasses shouldn't always share all characteristics of their parent class but will do
so with inheritance."*

Unreal's `AActor` → `APawn` → `ACharacter` → *your subclass* is exactly the model Rust does not
have, and ECS pushes further still — behaviour attaches to a **component set**, not to a type.

## Design patterns in Rust — mostly yes, often reshaped

Most Gang-of-Four patterns are implementable. Six are already in Phoenix, unnamed:

| pattern | its form here |
|---|---|
| **Command** | `Commands` — literally a queue of deferred operations applied at a sync point |
| **Factory** | `impl FromWorld for EnemyMaterials` — construct me, here is the World |
| **Observer** | messages: `PlayerDied`, `Hit`, each reader with its own cursor |
| **State** | `AppState` plus state-as-component-presence (`MoveTarget`, `Knockback`, `HitFlash`) |
| **Singleton** | a Resource — one global value per type, `init_resource` |
| **Strategy** | `match *mode` over `DeathMode`, selecting policy at the point of use |

Patterns that **dissolve** rather than translate: Iterator is in the language; Null Object is
`Option<T>`; Visitor is usually an `enum` plus `match`, because Rust has sum types and does not
need double dispatch; several creational patterns lose their reason to exist without constructors
to subvert.

Patterns that **fight ownership**: anything built on shared mutable back-references — classic
Observer holding `&mut` subjects, Composite with parent pointers, bidirectional graphs. The
idiomatic answer is an id or index into a store rather than a pointer, which is precisely what an
`Entity` is, and why Bevy models hierarchy as `ChildOf` / `Children` components instead of parent
references.

The Book's own caveat, worth keeping:

> *"Even though Rust is capable of implementing object-oriented design patterns, other patterns,
> such as encoding state into the type system, are also available in Rust... Object-oriented
> patterns won't always be the best solution in Rust due to certain features, like ownership, that
> object-oriented languages don't have."*

Lived example of the type-system alternative: `Dir3` cannot hold a non-unit vector, so
`Dir3::new(direction)` rejecting the zero vector *is* the "is the player moving?" check. A validity
invariant became a type instead of a guard someone has to remember.

## Related

- [[Bevy-Research-2026-09-25]] — why Bevy was chosen; ecosystem and costs
- [[Build-Log]] — the checkpoint-by-checkpoint record these lessons came from
- [[OOP-Foundations]] — superseded, Unreal-era object model
- [[Decisions]] ADR-008 — the switch to Rust + Bevy
