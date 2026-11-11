---
name: project-phoenix-state
description: Phoenix is in design/planning phase — no game code until the user says "go build"; key open decisions as of 2026-09-15
metadata:
  type: project
---

As of 2026-09-15 Phoenix is in **M0 pre-production, paper only**. The user explicitly said it is not time to build; zero game code exists.
The user hand-writes game code to learn (ADR-003) — Claude drafts research/plans/specs and reviews.

**Engine switched 2026-09-25 to Rust + Bevy 0.19 (ADR-008)** — the Unreal toolchain work and all Unreal-specific specs are superseded but kept. The user writes the code by hand in vim and wants to wire systems manually; Claude teaches and reviews, never builds unasked.

Decided 2026-09-15: **co-op is in v1 — friends can join** (ADR-007), so the 3-mode PIE matrix is a release gate and a
"Sessions and joining" spec is needed. Player death at 0 HP confirmed.

**2026-11-11 — own Rust game engine planned.** The user: *"I am planning to build own game engine with rust. I did once at the university, so we can proceed the plan paper."* Plan paper started at `04-production/Engine-Plan-2026-11-11.md` (research step waived by the user). ADR-005/008 had recorded the engine as a separate, later project. Paper only — no engine code until "go build". The university engine was not found on disk (UCI CS 112 = WebGL scene, CS 113 = Unity C# team game).

Open decisions awaiting the user (do not treat as decided):
- Engine D1: relation to phoenix — separate (A), phoenix moves onto it (B), or separate with a port of phoenix's slice as its first target (C, recommended).
- Engine D2: user answered *"B, Like vulkan"* (2026-11-11) — read as "write the GPU layer yourself on an explicit, Vulkan-kind API"; confirmation asked, because D1 got no separate answer. Sub-choice D2a: Vulkan via `ash` (needs Vulkan SDK + MoltenVK — NOT installed on this Mac) or Metal via `objc2-metal` (native, M1 Max). Recommended Vulkan.
- The university engine = `github.com/sjw2023/CS-112` (WebGL, 2018): hand-written GLSL, Phong-style lights, textures, skybox, trackball camera; no shadows, no ECS. Recorded in Engine-Plan §3.1.
- Hit-number throttle: how to stop Unreal dropping `MulticastHit` beyond 2 per object per update (Network-Protocol open question 1; recommended: raise `net.MaxRPCPerNetUpdate` + test).
- Attack design: the user wants **both** single-target and cleave — design options presented, choice pending.
- Co-op death: revive by a friend vs respawn, and any penalty.
- Approval of proposed names (Combat-Tech §15, Network-Protocol).
- Production Plan r2 (review fixes) is on hold until the design doc is expanded.

**Why:** the user redirected twice toward deeper design before planning or building.
**How to apply:** default to paper work (specs, research), keep every claim engine-source-verified, and never start code without "go build".

Related: [[reference-phoenix-vault]], [[feedback-phoenix-design-depth]]
