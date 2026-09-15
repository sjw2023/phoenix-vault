---
name: feedback-push-vault-changes
description: Standing permission — commit and push the Phoenix vault to GitHub whenever notes or the memory copy change
metadata:
  type: feedback
---

The user granted standing permission on 2026-09-15 ("1. Yes") to **commit and push the Phoenix Obsidian vault** to
https://github.com/sjw2023/phoenix-vault whenever vault notes change, and to re-sync the memory copy in
`99-claude-memory/` in the same push.

**Why:** they read the vault from other machines while using Remote Control; an unpushed change is invisible there.

**How to apply:** after a round of vault edits, run the link check, then one short commit (`docs : ...` prefix, no
Co-Authored-By footer per the user's global commit rules) and `git push`. Verify local HEAD equals remote HEAD. This
permission covers the **vault only** — never the game code repo `~/Workspace/phoenix`, which still needs an explicit ask.
The repo is public: never push secrets, tokens, internal hostnames or IPs.

Related: [[reference-phoenix-vault]], [[project-phoenix-state]]
