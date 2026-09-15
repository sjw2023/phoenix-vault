---
name: feedback-phoenix-design-depth
description: For Phoenix, the user wants full-detail feature design (OOP class design + server/multiplayer design) before plans or building
metadata:
  type: feedback
---

The user said the design doc was far too thin ("each feature of game is only described in 5 lines"), and that they lack OOP concepts and server design.
They want each feature as a full spec: a game-design note (`05-game-design/`) plus a technical note (`01-architecture/<Feature>-Tech.md`)
with CRC responsibilities, class diagram, ownership/lifetime, runtime sequence, what runs on server vs client, failure handling, tests.
Combat was the pilot (2026-09-15); remaining features follow the same template after the user reviews the pilot.

**Why:** they are learning game development from scratch and want to work "like pro game developers"; they delegated architecture calls
("you work as senior game developer and architecture") but naming/deletion approvals still apply.
**How to apply:** explain concepts in full with real engine code/source citations; run two independent reviewers (design + claim-verification)
on each spec and re-verify their findings before revising; quote web sources only from raw page text, never tool summaries.

Related: [[project-phoenix-state]], [[reference-phoenix-vault]]
