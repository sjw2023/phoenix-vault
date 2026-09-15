---
name: reference-phoenix-vault
description: Phoenix game project knowledge lives in the Obsidian vault, not the repo; where to start reading
metadata:
  type: reference
---

Phoenix (UE 5.8 C++ top-down ARPG, learning project) keeps all design, research, plans and decisions in the Obsidian vault at
`~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Phoenix/`. Start at `HOME.md`. The vault is a git repo pushed to **https://github.com/sjw2023/phoenix-vault** (PUBLIC, branch main, created 2026-09-15) — commit and push vault changes so the user can read them on other machines; never put secrets in it (current milestone + map of content).

Layout (numbered clusters, adopted 2026-09-14): `00-orientation/` (Design-Doc, Decisions = ADR log, Build-Log, Documentation-Framework),
`01-architecture/` (Architecture, OOP-Foundations, `<Feature>-Tech.md`), `02-build-steps/`, `03-research/` (`<Topic>-Research-<date>.md` + hub),
`04-production/` (Production-Plan), `05-game-design/` (`<Feature>.md`).

The repo `~/Workspace/phoenix` was reset to empty on 2026-09-14 (git + LFS only); the old scaffold is at tag `pre-reset`.
Engine source for verifying claims: `/Users/Shared/Epic Games/UE_5.8/Engine/Source/`.

Miro board "Phoenix" (created 2026-09-15 with the user's OK): https://miro.com/app/board/uXjVHndMfFs=/ — add all Phoenix diagrams here; the vault note is canonical and links to the widget.

A **copy** of this memory directory lives in the vault at `99-claude-memory/` (hyphenated filenames so Obsidian links resolve) and is pushed with the **public** repo — user's decision 2026-09-15, a copy rather than the RPS-style symlink. **The copy goes stale:** after changing any memory note, re-copy it into `99-claude-memory/` and commit + push with the vault. Because the repo is public, memory notes must never contain secrets, tokens, internal hostnames or IPs.

Related: [[project-phoenix-state]], [[feedback-phoenix-design-depth]]
