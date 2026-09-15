---
title: Git LFS — Research
date: 2026-09-14
status: research-draft
tags: [phoenix, research, git, tooling]
---

# Git LFS — Research

> [!info] **Scope.** Why git handles binary game assets badly, what Git LFS does about it, and
> what it costs. **Facts and mechanism only — no project decisions.** The decision to adopt it
> is recorded in [[Decisions]].

## Revision history

| rev | date | what changed |
|---|---|---|
| r1 | 2026-09-14 | First draft. Written while setting up version control for the empty project. |

## Evidence tiers used in this note

Following the house rule that every claim says how it was established:

| tag | meaning |
|---|---|
| **[measured]** | Ran it on this machine, numbers reproduced below |
| **[source]** | Read in the installed engine / git's own files |
| **[doc]** | Official documentation |
| **[NOT verified]** | Believed but unconfirmed — listed in §7 |

---

## 1. The problem, stated as a measurement

**[measured]** Same content, same one-line edit, committed ten times. The only variable is whether
the content sits in a plain text file or inside a compressed container — which is what a `.uasset`
is.

| | file on disk | `.git/` after 10 commits |
|---|---|---|
| **A** — raw text | 2.6 MB | **532 KB** |
| **B** — compressed container | **396 KB** | **3,564 KB** |

**File B is 6.5× smaller on disk and produced a repository 6.7× larger.**

Repo A stored ten versions of a 2.6 MB file in one fifth the space of a single copy. Repo B stored
ten versions of a 396 KB file in roughly nine full copies.

The smaller file made the bigger repo. Everything below explains why, and LFS is the response.

## 2. How git actually stores things

A common wrong model: *"git stores the differences between versions."* It does not.

Git stores **complete snapshots**. Every version of every file is written whole as an object named
by the hash of its contents. Ten commits touching one file produce ten complete blobs.

Git claws that back in a **separate, later step**: when packing objects it searches for objects that
*resemble* each other byte-wise and stores the similar ones as **deltas** — "take that other object
and apply these edits."

The crucial property: this is a **heuristic byte-similarity search**, not a semantic diff. It
succeeds or fails depending on whether two versions share long byte runs.

### 2.1 The packfile, read directly

**[measured]** `git verify-pack -v` on both repos from §1, showing size-in-pack per blob:

```
repoA (text)                      repoB (compressed)
  blob  416160   ← one full copy    blob  391515   ← full copy
  blob     129   ← delta            blob  391607   ← full copy
  blob     129                      blob  391633   ← full copy
  blob     129                      blob  391570   ← full copy
  blob     129                      blob  391626   ← full copy
  ...nine at ~129 bytes...          ...nine at ~391 KB...
```

**Text:** nine of ten versions stored as ~129 bytes each. Delta compression working perfectly.
**Compressed:** nine near-full copies. The delta search found nothing to exploit.

### 2.2 Why compression defeats delta compression

Compression builds a dictionary of repeated patterns as it scans the input. Change one line near
the start and every subsequent pattern match shifts — so the dictionary differs, so the output bytes
differ **from that point onward**.

A one-line *logical* edit therefore produces a globally different *byte* stream. Git compares
version 1 and version 2, finds almost no shared byte runs, and stores both whole. It is not being
naive; the exploitable redundancy genuinely is not there.

**Unreal's `.uasset` and `.umap` are exactly this shape** — serialized, compressed containers. So
are `.png`, `.fbx`, `.wav`, `.mp4`: all already-compressed formats.

## 3. The two facts that turn this into a real problem

1. **Git history is append-only.** Deleting a file in a later commit reclaims nothing — the bytes
   stay reachable from older commits and live in `.git/` permanently. Removing them means rewriting
   history, which changes every commit hash after the offending one.
2. **`git clone` downloads the entire history by default** — not the current state, everything ever
   committed.

Together: a 40 MB texture iterated 20 times costs ~800 MB, forever, and every clone pays it to
deliver one 40 MB file. Per asset it looks harmless; an ARPG has thousands. This is the mechanism
by which game repos become 30 GB clones.

## 4. What LFS does

LFS keeps large binaries **out of git's object store** and commits a small text stand-in instead.

What git stores in place of a 40 MB texture — about 130 bytes:

```
version https://git-lfs.github.com/spec/v1
oid sha256:4d7a214614ab2935c943f9e0ff69d22eafe70b1b3d8a90c5b2ff4e9f7cd318a26
size 41943040
```

Text. Successive versions of *that* delta-compress to nearly nothing — git is back in the regime
it excels at. Repo history stays small no matter how often the texture is revised.

### 4.1 The mechanism — clean and smudge filters

**[doc]** LFS is built on a long-standing git feature: per-file-type **filters** that run on the way
in and the way out. This is why it is invisible in daily use.

```
   you save a file
         │
    git add                  ← CLEAN filter
         │                     real bytes  → .git/lfs/objects/   (local store)
         │                     pointer text → git's object store
         ▼
   commit / push            ← git pushes the tiny pointer;
                              LFS separately uploads bytes to the LFS server

   ──────────────────────────────────────────────

   git checkout / clone     ← SMUDGE filter
         │                     sees a pointer, fetches bytes by their oid
         ▼                     writes the REAL file into the working tree
   the actual 40 MB file on disk
```

> [!note] Readable copy on the [Phoenix Miro board](https://miro.com/app/board/uXjVHndMfFs=/?moveToWidget=3458764683755799841); **this note is canonical.**

Two stores instead of one: git holds pointers and source; LFS holds bytes. `add` / `commit` /
`push` / `checkout` behave normally — the filters run underneath.

### 4.2 The `.gitattributes` rule, field by field

```gitattributes
*.uasset filter=lfs diff=lfs merge=lfs -text
```

| field | effect |
|---|---|
| `filter=lfs` | run the clean/smudge filters — **the core of it** |
| `diff=lfs` | do not attempt a byte diff in `git diff` |
| `merge=lfs` | do not attempt a line-based merge |
| `-text` | never apply line-ending conversion — it would corrupt binary data |

### 4.3 Where the bytes live

- **Locally:** `.git/lfs/objects/`, named by `oid` — a cache, separate from git's object store.
- **Remotely:** an LFS server, speaking its own HTTP API distinct from the git protocol. GitHub,
  GitLab, Gitea and Bitbucket all provide one; self-hosting is possible.

Because the stores are decoupled, a clone fetches only the LFS objects for the commit actually
checked out — not every version ever made. That alone fixes the clone-size problem.

## 5. What to track, and what not to

| | rule |
|---|---|
| **LFS** | Irreplaceable binaries — `.uasset` `.umap` `.fbx` `.png` `.tga` `.wav` `.mp4` |
| **Plain git** | All text — `.cpp` `.h` `.cs` `.ini` `.uproject`. §1 repo A proves git is *excellent* here. Routing text through LFS is strictly worse: readable diffs and blame are lost. |
| **`.gitignore`** | Anything regenerable — `Binaries/` `Intermediate/` `DerivedDataCache/` `Saved/`. Not LFS's job. |

**The rule:** LFS for irreplaceable binaries, `.gitignore` for regenerable output, plain git for text.

## 6. Failure modes

1. **`.gitattributes` must exist BEFORE the asset is first committed.** LFS only intercepts files
   matching a rule at `git add` time. Commit a texture first and add the rule later and that
   texture is a permanent full blob. Fixing it needs `git lfs migrate`, which rewrites history.
   *This is the reason version control was set up before the editor was ever opened.*
2. **A clone on a machine without git-lfs yields pointer files** — you open what should be a
   texture and find three lines of text. Not corruption; the smudge filter never ran. Install
   git-lfs, then `git lfs pull`.
3. **`git lfs install` is separate from installing the binary.** `brew install git-lfs` puts it on
   disk; `git lfs install` registers the filters in the user's git config. Both are required.
4. **Hosted LFS storage and bandwidth are metered** on free tiers, and art assets consume both
   faster than code. Irrelevant while the repo is local-only.
5. **`GIT_LFS_SKIP_SMUDGE=1`** clones pointers only — useful to get source fast and assets later.

### 6.1 What LFS does NOT do

It does not make binaries mergeable. Two people editing the same `.uasset` is still an unresolvable
conflict; one version wins. Teams handle this with **file locking** (`git lfs lock`). Solo, it never
arises.

## 7. NOT verified

| claim | why it is unverified | what would settle it |
|---|---|---|
| Current GitHub free LFS quota (storage + monthly bandwidth) | Not checked live; published figures change | Read GitHub's billing docs at the time a remote is added |
| Whether `git lfs migrate` is painless on a repo this small | Never needed — `.gitattributes` was in place from commit #1 | Only relevant if a rule is ever added late |
| Whether an actual UE `.uasset` behaves exactly like the §1 model | §1 used a gzip container as a faithful *model* of a compressed serialized asset, not a real `.uasset` | Re-run §1 against real `.uasset` files once the project has some |

## 8. The industry alternative, for context

Most large game studios do not use git for assets — they use **Perforce (Helix Core)**, built for
huge binary depots with centralized file locking, and integrated natively in Unreal as a source
control provider.

Not appropriate here: it means running a server, and its strengths solve *team* problems this
project does not have. Git + LFS is the standard solo/indie Unreal choice. Worth knowing the name
so Unreal documentation referencing Perforce can be read and skipped.

## 9. What was actually set up

**[measured]** On `~/Workspace/phoenix`, 2026-09-14, before the editor was ever opened:

| step | result |
|---|---|
| `brew install git-lfs` | git-lfs **3.8.0** |
| `git lfs install` | filters registered |
| `git init` | repo created |
| `.gitignore` | UE regenerables, clangd artifacts, macOS noise |
| `.gitattributes` | LFS rules for 17 binary extensions — **in place at commit #1** |
| safety commit `8f929e3` | tagged `pre-reset` — the pre-reset scaffold |
| reset commit `60f2021` | project emptied |

## Related

- [[Decisions]] — the adoption decision and its rationale
- [[Documentation-Framework]] — why this note is shaped as an Explanation
- [[00-Research-Hub]] — index of all research notes
