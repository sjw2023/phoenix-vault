# Claude project memory — copy

A **copy** of Claude Code's project memory for Phoenix, kept here so it can be read on other machines.

| | |
|---|---|
| **Canonical location** | `~/.claude/projects/-Users-joowon-Workspace-phoenix/memory/` on the development Mac — Claude reads and writes there |
| **This folder** | A snapshot copy, not a link. It goes stale until re-copied |
| **Last synced** | 2026-09-15 |
| **Filenames** | Hyphenated to match each note's `name:` slug, so `[[wiki-links]]` resolve in Obsidian. The canonical files use underscores |

## What is in it

- `MEMORY.md` — the index Claude loads at the start of every session in the Phoenix project.
- `reference-*.md` — where things are (vault, repo, Miro board, engine source).
- `project-*.md` — current project state and open decisions.
- `feedback-*.md` — how the maintainer wants the work done, and why.

These notes are written for Claude to re-read, not as project documentation. The project's actual knowledge — design,
research, plans, decisions — lives in the rest of this vault, starting at [[HOME]].

## This repository is public

Memory copied here must never contain secrets, tokens, internal hostnames or IP addresses.

## Re-syncing

After a memory note changes, copy it here under its hyphenated name, update `MEMORY.md`'s links to the hyphenated
filenames, and commit and push with the vault.
