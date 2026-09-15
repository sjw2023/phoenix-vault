---
name: project-phoenix-state
description: Phoenix is in design/planning phase — no game code until the user says "go build"; key open decisions as of 2026-09-15
metadata:
  type: project
---

As of 2026-09-15 Phoenix is in **M0 pre-production, paper only**. The user explicitly said it is not time to build; zero game code exists.
The user hand-writes game code to learn (ADR-003) — Claude drafts research/plans/specs and reviews.

Toolchain settled and tested: UE 5.8.2 + Xcode 26.6 + Metal toolchain component, macOS only (ADR-006 r2).

Decided 2026-09-15: **co-op is in v1 — friends can join** (ADR-007), so the 3-mode PIE matrix is a release gate and a
"Sessions and joining" spec is needed. Player death at 0 HP confirmed.

Open decisions awaiting the user (do not treat as decided):
- Hit-number throttle: how to stop Unreal dropping `MulticastHit` beyond 2 per object per update (Network-Protocol open question 1; recommended: raise `net.MaxRPCPerNetUpdate` + test).
- Attack design: the user wants **both** single-target and cleave — design options presented, choice pending.
- Co-op death: revive by a friend vs respawn, and any penalty.
- Approval of proposed names (Combat-Tech §15, Network-Protocol).
- Production Plan r2 (review fixes) is on hold until the design doc is expanded.

**Why:** the user redirected twice toward deeper design before planning or building.
**How to apply:** default to paper work (specs, research), keep every claim engine-source-verified, and never start code without "go build".

Related: [[reference-phoenix-vault]], [[feedback-phoenix-design-depth]]
