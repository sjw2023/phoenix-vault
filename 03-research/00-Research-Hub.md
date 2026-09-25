---
title: Phoenix — Research cluster hub
project: phoenix
tags: [phoenix, hub, research]
---

# Phoenix — Research cluster hub

Every research note for Phoenix lives here, regardless of which subsystem it is about. Collected in
one place so the learning trail is visible as a whole, and so it is obvious how many topics are
open at once.

A research note is **Explanation** in [[Documentation-Framework|Diátaxis terms]] — understanding-
oriented, facts and mechanism, no design decisions. Decisions go to [[Decisions]].

| topic | note | status | one-line finding |
|---|---|---|---|
| Git LFS — why binaries break git | [[Git-LFS-Research-2026-09-14\|Git LFS — Research]] | `research-draft` | Compressed assets defeat git's delta compression — **measured 6.7× repo bloat vs text for a smaller file**. LFS swaps the bytes for a ~130-byte pointer. Rules must exist *before* first commit. |
| Toolchain — can this Mac compile Unreal C++? | [[Toolchain-Research-2026-09-14\|Toolchain — Research]] | `research-draft` | **UE 5.5.4 + Xcode 26.6 cannot build (tested, exit 6).** No Xcode is inside both UE 5.5's range and Apple's support for macOS 26.6.2. Only UE 5.8 + Xcode 26.1.1 is documented as supported by both vendors, and it is untested. |
| Game production — how studios plan, vs the design doc | [[Game-Production-Research-2026-09-14\|Game Production — Research]] | `research-draft` | Milestones mean only what their **written exit criteria** say. Prototype the core loop first. **Gap G1:** pillar 1 (combat feel) is only checked at build step 9. |
| Unreal networking — from gameplay call to UDP | [[Unreal-Networking-Research-2026-09-15\|Unreal Networking — Research]] | `research-draft` | UDP packets ≤ 1024 bytes; bit-packed bunches on channels; engine-built acks and resends; reliable queue 512. **Iris is off by default in 5.8 — Phoenix uses legacy replication.** |
| Bevy — moving Phoenix to Rust | [[Bevy-Research-2026-09-25\|Bevy — Research]] | `research-draft` | ECS, **no editor yet** ("upcoming Bevy Editor"), breaking changes every release ("still in the experimentation phase"). Physics, navmesh and server-authoritative networking crates all exist on 0.19. |

**Five topics.**

## Open questions across the cluster

- ~~**Q0** — which Unreal Engine + Xcode combination to use.~~ **Decided:** UE 5.8 + Xcode 26.1.1, macOS only ([[Decisions]] ADR-006). Still blocking until the install is verified to compile. See [[Toolchain-Research-2026-09-14#6. Candidate combinations|Toolchain §6]].

- **Q1** — current GitHub free LFS quota (storage + monthly bandwidth). Blocks nothing until a
  remote is added. See [[Git-LFS-Research-2026-09-14#7. NOT verified|Git LFS §7]].

## Naming

`<Topic>-Research-<YYYY-MM-DD>.md`, matching the convention used in the RPS and Vault-API vaults.
