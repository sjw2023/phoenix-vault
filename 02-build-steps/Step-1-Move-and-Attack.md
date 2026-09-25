---
type: build-step-spec
project: Phoenix
step: 1
status: not-started
updated: 2026-08-03
engine: Unreal Engine 5 (C++)
tags: [project/phoenix, gamedev, spec, step-1]
---

# Phoenix — Step 1: Move & Attack

> [!warning] **Superseded 2026-09-25 — written for Unreal Engine.** Phoenix moved to Rust + Bevy ([[Decisions]] ADR-008). Kept as history; the reasoning may still be useful, the mechanisms are not. Engine-independent design lives in [[Design-Doc]] and [[Combat]].


Technical spec for the first build step from the [[Design-Doc|build order]], for **Unreal Engine 5, C++, 3D top-down**: a controllable character in an empty level that click-to-moves and does a placeholder attack, viewed from a fixed top-down camera (like Path of Exile). The real goal is **learning Unreal's Gameplay Framework** — Actor/Pawn/Controller/GameMode, components, Enhanced Input, and navigation. The gameplay is deliberately thin. Follows [[Architecture]] and [[Coding-Conventions]].

## Prerequisites (one-time setup)
- Unreal Engine 5 installed + a C++ toolchain: **Visual Studio 2022** (Windows) or **Xcode** (macOS). On macOS, since your project folder is `/Users/joowon/Workspace/phoenix`, open `Phoenix.uproject` there.
- First open: Unreal will prompt to **generate project files** and **compile the C++ module** — say yes. If the engine-version prompt appears, associate with your installed UE5 version (the `.uproject` will list `5.8` — see [[Decisions]] ADR-006).

## Goal
A 3D character that:
- moves to where you left-click on the ground (top-down navigation),
- plays a placeholder attack on a second input,
- is viewed by a fixed top-down camera that follows it,
running at a stable frame rate in an empty level with a floor. No enemies, no stats wiring, no damage yet.

## Explicitly out of scope for step 1
Enemies, health/damage, wiring the `UStatsComponent`, real animations, loot, UMG HUD. Those are later steps. Resist adding them.

## Classes (all C++ base, Blueprint subclass for tuning)
```
APhoenixCharacter : ACharacter          // Player/  — capsule + mesh + camera boom
APhoenixPlayerController : APlayerController   // Core/ — click-to-move input
APhoenixGameMode : AGameModeBase         // Core/  — sets DefaultPawn + PlayerController (to be written — project reset to empty 2026-09-14)
```
Create Blueprint subclasses in the editor: `BP_PhoenixCharacter` (assign a mesh), `BP_PhoenixGameMode` (set `DefaultPawnClass = BP_PhoenixCharacter`, `PlayerControllerClass = APhoenixPlayerController`). Point the level / Project Settings → Maps & Modes at `BP_PhoenixGameMode`.

## Camera (top-down)
On `APhoenixCharacter`, in the constructor:
```cpp
SpringArm = CreateDefaultSubobject<USpringArmComponent>(TEXT("SpringArm"));
SpringArm->SetupAttachment(RootComponent);
SpringArm->TargetArmLength = 1200.f;
SpringArm->SetRelativeRotation(FRotator(-60.f, 0.f, 0.f)); // pitch down
SpringArm->bDoCollisionTest = false;
SpringArm->bInheritPitch = SpringArm->bInheritYaw = SpringArm->bInheritRoll = false; // fixed angle

TopDownCamera = CreateDefaultSubobject<UCameraComponent>(TEXT("TopDownCamera"));
TopDownCamera->SetupAttachment(SpringArm, USpringArmComponent::SocketName);

// Character faces movement direction, controller rotation doesn't spin the camera:
bUseControllerRotationYaw = false;
GetCharacterMovement()->bOrientRotationToMovement = true;
```

## Input: Enhanced Input (UE5 standard)
Enhanced Input uses assets, not `.ini`. In the editor create:
- **`IMC_Default`** (Input Mapping Context)
- **`IA_MoveClick`** (Digital/bool) → mapped to **Left Mouse Button**
- **`IA_Attack`** (Digital/bool) → mapped to **Right Mouse Button** (or a key)

Add `IMC_Default` to the local player subsystem in `BeginPlay`, then bind the actions in `SetupInputComponent`.

## Click-to-move (navigation)
Top-down click-to-move uses the **navigation system**, so the level needs a **NavMeshBoundsVolume** covering the floor (drop one in, press **P** to visualize). On click: deproject the cursor to the world, then move there.

`APhoenixPlayerController` sketch:
```cpp
void APhoenixPlayerController::BeginPlay()
{
    Super::BeginPlay();
    bShowMouseCursor = true;
    if (UEnhancedInputLocalPlayerSubsystem* Subsys =
        ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(GetLocalPlayer()))
    {
        Subsys->AddMappingContext(DefaultContext, 0); // DefaultContext = IMC_Default (UPROPERTY, set in BP)
    }
}

void APhoenixPlayerController::OnMoveClick()
{
    FHitResult Hit;
    if (GetHitResultUnderCursor(ECC_Visibility, false, Hit))
    {
        UAIBlueprintHelperLibrary::SimpleMoveToLocation(this, Hit.ImpactPoint);
        // (SimpleMoveToLocation uses the nav mesh + pathfinding under the hood.)
    }
}

void APhoenixPlayerController::OnAttack()
{
    // Step 1: placeholder — log + a quick visual tell later.
    UE_LOG(LogPhoenix, Log, TEXT("Attack"));
}
```
Bind in `SetupInputComponent` via `UEnhancedInputComponent::BindAction(MoveClickAction, ETriggerEvent::Started, this, &APhoenixPlayerController::OnMoveClick)` (and similarly `IA_Attack`).

> Note: `SimpleMoveToLocation` needs `NavigationSystem` + `AIModule`. Add `"NavigationSystem"`, `"AIModule"` to `Phoenix.Build.cs` dependencies when you implement this (the `.Build.cs` currently lists Core/Engine/InputCore/EnhancedInput).

## Acceptance criteria (definition of done)
- [ ] Left-clicking the ground moves the character there via pathfinding and it stops cleanly.
- [ ] Attack input fires `OnAttack()` — visible in the Output Log.
- [ ] Top-down camera holds a fixed angle and follows the character.
- [ ] Character orients to its movement direction.
- [ ] Movement is framerate-independent (CharacterMovement handles this).
- [ ] Code matches [[Coding-Conventions]] (prefixes, `UPROPERTY`, Enhanced Input, correct folders).
- [ ] A comment marks where `MoveSpeed` will later read from `UStatsComponent`.

## Learning checklist (the real point)
By the end you should be able to explain: Actor vs Component; Pawn vs Controller vs GameMode (who possesses whom); how Enhanced Input maps devices → actions → bound functions; what a NavMeshBoundsVolume does and why click-to-move needs it; and how a SpringArm + Camera makes a top-down view. If any are fuzzy, nail them before step 2.
