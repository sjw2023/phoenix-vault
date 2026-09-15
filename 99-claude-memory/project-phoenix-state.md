---
name: project-phoenix-state
description: Phoenix is in design/planning phase — no game code until the user says "go build"; key open decisions as of 2026-09-15
metadata:
  type: project
---

As of 2026-09-15 Phoenix is in **M0 pre-production, paper only**. The user explicitly said it is not time to build; zero game code exists.
The user hand-writes game code to learn (ADR-003) — Claude drafts research/plans/specs and reviews.

Toolchain settled and tested: UE 5.8.2 + Xcode 26.6 + Metal toolchain component, macOS only (ADR-006 r2).

Open decisions awaiting the user (do not treat as decided):
- Multiplayer scope ("option B"): v1 single-player, architecture multiplayer-correct — whether the 3-mode PIE matrix is a v1 release gate is unanswered; no ADR yet; Design-Doc §3 still says multiplayer out of scope.
- Production Plan r2 (review fixes) is on hold until the design doc is expanded.
- Combat pilot specs r2 (Combat.md, Combat-Tech.md, OOP-Foundations.md) await the user's read: new-name approvals, player-death rule, single-target vs cleave.

**Why:** the user redirected twice toward deeper design before planning or building.
**How to apply:** default to paper work (specs, research), keep every claim engine-source-verified, and never start code without "go build".

Related: [[reference-phoenix-vault]], [[feedback-phoenix-design-depth]]
