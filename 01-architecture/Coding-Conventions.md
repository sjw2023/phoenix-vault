---
type: conventions
project: Phoenix
updated: 2026-08-03
engine: Unreal Engine 5 (C++)
tags: [project/phoenix, gamedev, conventions, style]
---

# Phoenix — Coding Conventions

> [!warning] **Superseded 2026-09-25 — written for Unreal Engine.** Phoenix moved to Rust + Bevy ([[Decisions]] ADR-008). Kept as history; the reasoning may still be useful, the mechanisms are not. Engine-independent design lives in [[Design-Doc]] and [[Combat]].


The C++ style guide for Phoenix, following Epic's Unreal coding standard. Goal: consistency so future code moves fast. The built `Stats/` code is the reference implementation. See [[Architecture]] for structure.

## Class prefixes (mandatory — the compiler-adjacent tooling relies on them)
Unreal encodes a type's category in a one-letter prefix:

| Prefix | Meaning | Example |
|---|---|---|
| `U` | `UObject`-derived (incl. components) | `UStatsComponent` |
| `A` | `AActor`-derived | `APhoenixCharacter` |
| `F` | plain struct / non-UObject class | `FStatModifier` |
| `E` | enum | `EModifierType` |
| `T` | template | `TArray`, `TMap` |
| `I` | interface | `IInteractable` |
| `b` | boolean variable | `bIsDead`, `bDirty` |

Get these right — the reflection system and headers assume them.

## Files
- **One class per header**, file named after the class **without** the prefix: `UStatsComponent` → `StatsComponent.h/.cpp`; `APhoenixGameMode` → `PhoenixGameMode.h/.cpp`.
- `#pragma once` at the top of every header.
- The `.generated.h` include is **always last** in a reflected header's include list.
- Group source by system under `Source/Phoenix/<System>/` (see [[Architecture]]).

## Naming
- **Types, functions, and variables: `PascalCase`** — `GetValue`, `MoveSpeed`, `RemoveModifiersFromSource`. (Note: Unreal uses PascalCase for *everything*, unlike Godot's snake_case — retrain the fingers.)
- **No Hungarian `m_`** on members; just PascalCase. Booleans take the `b` prefix: `bDirty`.
- **Constants / enum values: PascalCase** — `EModifierType::Flat`.
- **FName keys** for stats/ids: string literals auto-convert (`"physical_damage"`). FName is the interned-key type — fast compares — and is the C++ analog of Godot's StringName.

## Reflection macros — use them on anything the engine/Blueprint touches
- `UCLASS()` / `USTRUCT()` / `UENUM()` above the type, with `GENERATED_BODY()` inside.
- `UPROPERTY(...)` on member variables that need editor exposure, replication, or **GC tracking** (see memory below).
- `UFUNCTION(BlueprintCallable, Category="...")` on functions callable from Blueprint.
- Always set a `Category` on exposed properties/functions so the Details panel stays organized.
- Expose tunables with `UPROPERTY(EditAnywhere, BlueprintReadWrite)` — the `@export` equivalent.

## Memory & pointers (the big difference from plain C++)
- **Never `new`/`delete` `UObject`s.** Create with `NewObject<T>()`, actors with `SpawnActor<T>()`. The **garbage collector** owns them.
- A `UObject*` member **must** be a `UPROPERTY()` (or `TObjectPtr<T>`, preferred in UE5) or the GC may collect it out from under you.
- Non-owning refs: `TWeakObjectPtr<T>`. Containers: `TArray`, `TMap`, `TSet` (not `std::vector`/`std::map`).
- Non-UObject heap objects: `TSharedPtr` / `TUniquePtr`.
- Pass containers by `const T&`; return small structs by value.

## Delegates over references
Prefer a multicast delegate (often on the event-bus subsystem) to a direct pointer between systems. `DECLARE_DYNAMIC_MULTICAST_DELEGATE(...)` for Blueprint-assignable events; bind in `BeginPlay`.

## Strings
- `FName` — interned keys/ids (stats, tags). Cheap compare.
- `FString` — mutable working text.
- `FText` — user-facing, localizable display text (UI labels).
Don't use `std::string`.

## Const, safety, logging
- `const`-correct: mark read-only methods `const` (`GetBase(...) const`).
- Validate with `check()` (fatal) / `ensure()` (non-fatal, continues) / `IsValid(Ptr)` before deref.
- Log with `UE_LOG(LogPhoenix, Log, TEXT("..."))` — define a `LogPhoenix` category; never `printf`. Wrap literals in `TEXT("...")`.

## Comments
- `/** ... */` doc comments on classes, `UPROPERTY`s, and public functions (surface in the editor tooltip).
- Every file's primary type gets a short `/** */` header describing its role (example: `StatsComponent.h` at git tag `pre-reset`).
- Comment the **why**; mark future seams: `// TODO(step 3): drive MoveSpeed from UStatsComponent`.

## Layout within a class
1. `UCLASS()` + class decl (`public` first in Unreal style)
2. constructor / `BeginPlay` / lifecycle
3. public API (with `UFUNCTION` where exposed)
4. `protected`
5. `private` members last (`UPROPERTY()` data, then helpers)

## Data-driven reminder
Per [[Architecture]]: new content = a **Data asset** (`UPrimaryDataAsset` / `UDataTable` row), not a new `UCLASS`. If a new `AFoo : ABar` is about to describe a *variant of existing content*, stop — it should be data + components.
