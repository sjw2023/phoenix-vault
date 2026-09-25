---
title: Network Protocol — Phoenix message catalog
date: 2026-09-15
status: draft-awaiting-comment-step
revision: r2
tags: [phoenix, architecture, networking, protocol, reference]
engine: Unreal Engine 5.8.2
---

# Network Protocol

> [!warning] **Superseded 2026-09-25 — written for Unreal Engine.** Phoenix moved to Rust + Bevy ([[Decisions]] ADR-008). Kept as history; the reasoning may still be useful, the mechanisms are not. Engine-independent design lives in [[Design-Doc]] and [[Combat]].


> [!info] **What this is.** The **one list** of every message Phoenix sends over the network: client commands, server
> events and replicated state, across all features. **Reference form** in [[Documentation-Framework]] terms — a
> lookup table, not an explanation. How the engine carries these messages: [[Unreal-Networking-Research-2026-09-15]].
>
> A readable table copy is on the [Phoenix Miro board](https://miro.com/app/board/uXjVHndMfFs=/?moveToWidget=3458764683744620161); **this note is canonical.**
>
> **Rule for every future feature spec:** its technical spec adds its rows here, and a feature is not accepted
> until its rows pass the §1 rules.

> [!note] **Planning only.** Names marked **proposed** still need approval ([[Combat-Tech#15. Open questions|Combat-Tech §15]]).
> Evidence: **[engine]** = UE 5.8.2 source; **research §N** = sections of [[Unreal-Networking-Research-2026-09-15]]; **[design]** = a Phoenix decision,
> open to review; **(H)** = starting value to tune.

## Revision history

| rev | date | what changed |
|---|---|---|
| r1 | 2026-09-15 | First draft. Combat rows from [[Combat-Tech]] r2. |
| **r2** | **2026-09-15** | After an independent review, every finding re-checked in source. **P6 replaced:** r1 dropped commands over a rate limit, which could leave a released attack button still attacking. Commands are now never dropped. **Added:** the engine's cap of 2 unreliable broadcasts per update (§3), ground-click cancel during recovery, `ClientSerial`, team and max-health rows, engine dependencies, `WithValidation` warning, P11, traffic assumptions. **Corrected:** P3, P5, P8, P9 reasons. See §10. |

---

## 1. Rules every message must pass

| # | rule | why |
|---|---|---|
| P1 | **Clients send intent, never results.** "I clicked this enemy", never "I dealt 40 damage". | The server decides outcomes — [[OOP-Foundations#7.2 The five rules\|OOP-Foundations §7.2]] |
| P2 | **Every client → server command has a server-side check** listed in §2, handled inside the command's implementation: ignore and log. **Do not use `WithValidation` for gameplay rejection.** | A failed validation **disconnects the client** — [[Unreal-Networking-Research-2026-09-15#8.5 A failed RPC validation disconnects the client\|research §8.5]]. Reserve it for input no honest client can produce |
| P3 | **Reliable only for rare, discrete commands** (a click, a press, a release). **Never per frame.** | A lost reliable call holds back later reliable calls on the same channel until it is resent, and two per-channel limits of 512 close the connection when reliable calls pile up behind loss — [[Unreal-Networking-Research-2026-09-15#6. Reliable versus unreliable — how resending works\|research §6]] |
| P4 | **Unreliable only for things safe to lose** — a lost message may cost a visual, never game state. | Unreliable is lost with its packet, **and unreliable broadcasts beyond 2 per function per object per update are dropped by the sender** — [[Unreal-Networking-Research-2026-09-15#8.4 Unreliable multicasts are queued and capped per update\|research §8.4]] |
| P5 | **State that must arrive is a replicated property, not an event.** Only its **latest** value is guaranteed; if every transition matters, carry a serial. | Properties are sent on change, on channel open and again after loss, always as the current value — [[Unreal-Networking-Research-2026-09-15#8. Property replication — two systems, one is the default\|research §8.1]] |
| P6 | **Commands are never dropped. Every command is coalescing or idempotent:** a *set* command applies its latest value; a *request* replaces the pending request; a *cancel* is safe to repeat. Per-player counters exist **only to log** abuse. | Dropping runs after the engine has already received and acknowledged the call, so it protects nothing — and dropping a release or cancel leaves the server in the wrong state [design] |
| P7 | **A replicated C++ property's reaction runs on both paths** (server and `OnRep_`). | RepNotify runs only on receivers for C++ properties — [[Unreal-Networking-Research-2026-09-15#8.2 The client path\|research §8.2]] |
| P8 | **Positions and directions are sent quantized** (`FVector_NetQuantize` family) unless precision requires otherwise. | Quantized vectors cost a 7-bit header plus only the bits each value needs, instead of three full values — [[Unreal-Networking-Research-2026-09-15#9.3 Sending fewer bits — quantized vectors\|research §9.3]] |
| P9 | **Object parameters may arrive as null.** The receiver checks every object pointer. | By default an RPC with an unresolved object runs immediately with a null parameter — [[Unreal-Networking-Research-2026-09-15#7. Object references — NetGUIDs\|research §7]] |
| P10 | **Names follow the engine's kind:** `Server…`, `Multicast…`, `Client…` for RPCs; `OnRep_<Property>` for notifies. | One glance tells direction |
| P11 | **Every RPC body is correct without a network hop.** In Standalone and on a listen host, a local player's `Server…` call and the server's `Multicast…` call run locally and immediately; nothing may assume a round trip. | Standalone is *"Still considered a server because it has all server functionality"* (`EngineBaseTypes.h:980`) — [[OOP-Foundations#7.1 Why this costs almost nothing in single-player\|OOP-Foundations §7.1]] |

## 2. Client → server commands

| command | owner | parameters | reliability | sent when | server check | if the check fails | repeated calls | feature |
|---|---|---|---|---|---|---|---|---|
| `ServerRequestAttack` | `APhoenixPlayerController` | `AActor* Target`, `uint8 ClientSerial` *(proposed)* | Reliable | each enemy click | controller has a pawn · `Target` not null and is an `APhoenixCharacter` · target alive · target on the other team · attacker alive | ignore · `Verbose` log with the `EPhoenixHitCheck` value | **replaces** the pending request — latest wins | [[Combat-Tech#5.1 Player clicks an enemy\|Combat]] |
| `ServerSetHoldAttack` | `APhoenixPlayerController` | `bool bHeld` | Reliable | mouse press and release only | `bHeld == false`: **always accepted**. `bHeld == true`: attacker alive, a current target exists | ignore · `Verbose` log | **last value wins** | [[Combat-Tech#5.2 Hold to attack, and cancelling\|Combat]] |
| `ServerCancelAttack` | `APhoenixPlayerController` | none | Reliable | **any ground click while the client believes an attack is active or pending** — approach, windup **or recovery** | none — always safe | — | **idempotent** | [[Combat-Tech#5.2 Hold to attack, and cancelling\|Combat]] |

**Abuse logging [design]:** each command is counted per player over a sliding one-second window on **unscaled** time
(`FPlatformTime::Seconds()`, unaffected by pause or time dilation). Above **10 per second (H)**, a `Warning` naming the
player is logged at most once a minute. **Nothing is dropped.**

**Flood protection** is the engine's job: `[GameNetDriver RPCDoSDetection]` exists (`BaseEngine.ini:1924`) and is **off**
by default (`bRPCDoSDetection=false`). Whether to enable it is open question 2 — its tier behaviour, including
kicking, is not yet verified ([[Unreal-Networking-Research-2026-09-15#13. NOT verified\|research §13]]).

**Why `ClientSerial`:** the owning client plays its windup early (P11-safe, cosmetic). When the server's
`AttackRep` later replicates, the client must know **which** windup it already played. The server copies the
request's `ClientSerial` into `AttackRep.SourceClientSerial`; server-driven windups (hold loop, AI) carry 0.

**Movement is not a Phoenix command.** The owning client moves its own character; the engine's movement RPCs carry it
(§5).

## 3. Server → clients events

| event | owner | parameters | kind | reliability | sent when | receivers use it for | if lost **or throttled** | feature |
|---|---|---|---|---|---|---|---|---|
| `MulticastHit` | `UHealthComponent` | `FPhoenixHitCosmetic { int32 Dealt; bool bCrit; AActor* Instigator }` | NetMulticast | Unreliable | once per resolved hit | damage number, flash, hit-stop, sound | a missing number or flash; health is still correct (§4) | [[Combat-Tech#6.2 RPCs\|Combat]] |
| `MulticastWhiff` | `UCombatComponent` | none | NetMulticast | Unreliable | once per miss | whiff animation | a miss looks like nothing happened | [[Combat-Tech#6.2 RPCs\|Combat]] |

> [!warning] **The engine throttles these.** **[engine]** Unreliable multicasts are **queued until the object's next
> replication update**, and **more than 2 calls of the same function on the same object per update are dropped by the
> sender** (`net.MaxRPCPerNetUpdate` = 2) — [[Unreal-Networking-Research-2026-09-15#8.4 Unreliable multicasts are queued and capped per update\|research §8.4]].
>
> **Where this bites:** several enemies hitting **one** player — every hit is a `MulticastHit` on that player's
> `UHealthComponent`. If more than 2 land within one update, the extra damage numbers never reach any client. Because
> every enemy of a type shares the same attack timing ([[Combat#4.1 Phases|Combat §4.1]]), a pack that starts attacking
> together may keep its hits clustered **[NOT verified — measure in M1]**. See open question 1.

## 4. Replicated state

**[engine]** Replication conditions (`CoreUObject/Public/UObject/CoreNetTypes.h:18-21`): `COND_None` — *"will send
anytime it changes"*; `COND_InitialOnly` — *"will only attempt to send on the initial bunch"*; `COND_OwnerOnly` —
*"will only send to the actor's owner"*; `COND_SkipOwner` — *"send to every connection EXCEPT the owner"*.

| owner | property | type | notify | condition | changes when | used for | feature |
|---|---|---|---|---|---|---|---|
| `APhoenixCharacter` | `Team` *(proposed)* | `EPhoenixTeam` | — | `COND_InitialOnly` | set once at spawn | client pick rule: target team Monsters only | Combat |
| `UHealthComponent` | `Health` | `float` | `OnRep_Health(float OldHealth)` | `COND_None` | damage, healing, regeneration | health bars | [[Combat-Tech#6.1 Replicated state\|Combat]] |
| `UHealthComponent` | `MaxHealth` | `float` | `OnRep_MaxHealth` → `OnHealthChanged` *(proposed)* | `COND_None` | stats change (from M2) | health bars | Combat |
| `UHealthComponent` | `bIsDead` | `bool` | `OnRep_IsDead` → `HandleDeath` | `COND_None` | death; a new pawn on respawn | collision off, death feedback | Combat |
| `UHealthComponent` | `DeathServerTime` | `double` | — | `COND_None` | death | skip the death burst for an old death | Combat |
| `UCombatComponent` | `AttackRep` | `FPhoenixAttackRep { EPhoenixAttackPhase Phase; uint8 AttackSerial; uint8 SourceClientSerial }` | `OnRep_Attack` | `COND_None` | every phase change | windup tell; correct pose for late arrivals | Combat |

**Not replicated, on purpose:** `UHealthComponent::LastKiller`. Clients' `OnDied` receives a **null** killer; anything
that needs the killer (loot, XP, kill credit) runs on the server only.

**Constraints:**
- `bIsDead` and `DeathServerTime` stay on the **same object**, so `OnRep_IsDead` can read `DeathServerTime` —
  notifies run after all received properties are applied (`RepLayout.cpp:4661`).
- `AttackRep` is `COND_None`, not `COND_SkipOwner`: the owner still needs the server's phase, e.g. when an approach
  gives up after 3 s.
- Per P5, a phase that begins and ends between two updates may never be seen by a client — the windup tell must not
  depend on seeing every phase.

## 5. Engine-managed replication Phoenix relies on

| what | carried by | Phoenix dependency |
|---|---|---|
| Character position and velocity | engine character movement; client uploads via `ServerMove` / `ServerMovePacked` (named in `BaseEngine.ini:1931-1932`) | click-to-move, approach, kill knockback |
| Which controller owns which pawn | engine possession | `Server…` RPC ownership rule |
| Actor spawn and destruction | actor channel open and close ([[Unreal-Networking-Research-2026-09-15#5. Channels\|research §5]]) | enemies, corpses, respawned players |
| Object pointers as NetGUIDs | `UPackageMapClient` ([[Unreal-Networking-Research-2026-09-15#7. Object references — NetGUIDs\|research §7]]) | every `AActor*` parameter |
| Server world time on clients | `AGameStateBase` replicated time, updated every 0.1 s (`GameStateBase.cpp:36`) | `DeathServerTime` freshness check (`Age < 0.5`) |

P3 and P6 apply to **Phoenix-authored** RPCs; the engine's movement RPCs follow their own rules.

## 6. Traffic estimate — combat

Message **rates** follow from the design; **bytes** are not estimated, because overhead has not been measured.

**Assumed [design]:** 3 players and 10 enemies all attacking at [[Combat#4.1 Phases|Combat §4.1]]'s 2.5 attacks per second,
every attack hitting.

| message | generated per second, whole server | reliable? |
|---|---|---|
| `MulticastHit` | ≤ 13 × 2.5 = **32.5** — some may be throttled (§3) | no |
| `Health` changes | ≤ **32.5**, coalesced per actor per update | state |
| `AttackRep` changes | between 13 × 2.5 × **2** = **65** (hold loop: windup → recovery → windup) and × **4** at a lower rate (start from idle: idle → approach → windup → recovery → idle) | state |
| `ServerRequestAttack` | human clicks — typically well under 10 per player | yes |
| `ServerSetHoldAttack` / `ServerCancelAttack` | human presses | yes |

**Assumptions stated:**
- Rates count messages **generated**. Messages on the wire ≈ rate × relevant remote connections (2 in mode L, 2 in mode C).
- **No health regeneration.** Player regeneration (from M2) must change `Health` in coarse steps, or every regenerating
  player adds up to one `Health` change per network update.
- `NetUpdateFrequency` defaults to 100 per second but may be capped by the server's network tick — not verified.

**What to measure in M1:** actual bytes per second per client, and how many `MulticastHit` calls are throttled, in
**Play As Client** with network emulation on. The measuring tool is **[NOT verified]**.

## 7. Adding a feature's messages — checklist

- [ ] Direction, owner class, parameters with types
- [ ] **Command:** server check in the implementation, not `WithValidation` (P2); coalescing or idempotent on repeat (P6)
- [ ] **Event:** why losing **or throttling** it is harmless, or why it is state instead (P4, P5)
- [ ] **State:** notify, both-path handler, replication condition, what happens if an intermediate value is skipped (P5, P7)
- [ ] Reliability justified against P3
- [ ] Correct with no network hop (P11)
- [ ] Rows added to the §6 traffic estimate if frequent
- [ ] A matrix test in mode **C** (server with no local player)

## 8. NOT verified

| claim | why | what would settle it |
|---|---|---|
| Same-type enemies keep their hits clustered frame to frame | Inferred from shared timings | M1: log impact frame numbers per target |
| The 10-per-second logging threshold separates abuse from normal play | Design value (H) | M1: log peak per-player rates |
| Byte cost of each message | Overhead not measured | Measure in M1 (§6) |
| Which tool measures per-message bandwidth | Not researched | A short research pass before M1's measurement |
| What engine RPC flood detection does at each tier | Only configuration read | [[Unreal-Networking-Research-2026-09-15#13. NOT verified\|research §13]] |

## 9. Open questions

1. **How to stop the engine throttling `MulticastHit`** (§3) — decide before M1:
   - **(a) Raise `net.MaxRPCPerNetUpdate`** in `DefaultEngine.ini`. One line. Global: applies to every unreliable
     multicast in the game. At Phoenix's scale (3 players, packs of 10) the extra traffic is small.
   - **(b) Jitter enemy windups** by a small random offset (±0.03 s (H)) so hits stop clustering. Reduces the chance;
     does not remove the cap. Changes [[Combat#4.1 Phases|Combat §4.1]] timing.
   - **(c) Batch hits:** one `MulticastHits(TArray<FPhoenixHitCosmetic>)` per component per update. Deterministic,
     but needs a verified hook that runs once per replication update, and arrays cost more bits.
   - **(d) Replicated ring buffer** of the last few hits with a serial, as state. Not throttled, and resent after loss;
     costs a property update per hit.

   **Recommendation: (a) for M1, plus a matrix row** — 6 enemies hit one player in mode C; the client shows every number
   — and the M1 measurement of how many calls would have been throttled at the default. Move to (d) only if that
   measurement shows real traffic cost.
2. **Enable engine RPC flood detection?** Recommended only after its tiers are read (research §13).
3. **Approve names** — `ClientSerial`, `SourceClientSerial`, `Team`, `OnRep_MaxHealth` join the pending list in
   [[Combat-Tech#15. Open questions|Combat-Tech §15]].

## 10. Changes from r1

| r1 | r2 | evidence |
|---|---|---|
| P6: drop calls beyond 10 per second | **Never drop; coalescing or idempotent; count only to log** | Dropping happens after the engine received and acknowledged the call; a dropped `SetHold(false)` leaves the server holding |
| `ServerSetHoldAttack` required "attacker alive" | **`false` always accepted** | A dead player's release must still clear hold |
| `ServerCancelAttack` on ground click in approach or windup | **Also during recovery** | [[Combat#4.2 Rules\|Combat §4.2]] rule 5 — otherwise the server runs a click the player walked away from |
| No correlation between an early client windup and the server's `AttackRep` | **`ClientSerial` → `SourceClientSerial`** | Owner could not tell which windup it already played |
| "If lost: one missing number" | **Lost or throttled**; open question 1 | `NetDriver.cpp:3400,3453`; `DataReplication.cpp:38-42,2324-2331` |
| P3 reason: "block later reliable calls until acknowledged" | **Lost reliable call holds back later ones until resent; two separate 512 limits** | `DataChannel.cpp:640-696, 1414-1445` |
| P5 cited no mechanism | **Latest value only; NAK marks properties for resend** | `DataReplication.cpp:888-925` |
| P8 "up to 20 bits per component" | **7-bit header + bits the value needs** | Header comments stale: `NetSerialization.h:207-211`, `QuantizedVectorSerialization.cpp:90-104` |
| P9 cited no mechanism | **Unresolved objects run as null by default** | `DataReplication.cpp:44-49` |
| No `WithValidation` guidance | **P2: never for gameplay rejection** | Epic: *"the invoking client is disconnected"* |
| — | **P11 no network hop** | Standalone runs RPCs locally |
| Missing `Team`, `MaxHealth` notify, killer on clients, server time, movement RPCs | **Added** (§4, §5) | Combat relies on each |
| "× 3 phase changes"; rates unqualified | **2–4 with reasons; generated vs wire; no regeneration** | Review arithmetic check |

## Related

- [[Unreal-Networking-Research-2026-09-15]] · [[Combat-Tech]] · [[OOP-Foundations]] · [[Architecture]]
