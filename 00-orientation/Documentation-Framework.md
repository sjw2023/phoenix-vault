---
title: Documentation Framework — how notes in this vault are shaped
date: 2026-09-14
status: active
tags: [phoenix, orientation, meta, documentation]
---

# Documentation Framework

> [!info] **What this is.** The shapes a technical document can take, which one to reach for, and
> templates for each. Written because "what goes in a tech doc?" has a real answer and it is not
> the essay structure taught in school.

## 1. Start from what you already know — and where it stops working

The academic essay frame is:

```
Introduction (thesis)  →  Body 1 (evidence)  →  Body 2 (evidence)  →  Conclusion (wrap-up)
```

That frame is built for one job: **persuading a reader who starts at the top and reads to the
bottom, once.** The thesis is a claim, the body paragraphs are argument, the conclusion closes it.

Almost no technical document is read that way.

A technical document is **navigated, not read.** Its reader arrives mid-way, already annoyed,
with a specific need — *"how do I make click-to-move work?"*, *"what does this parameter do?"*,
*"why on earth is it built like this?"* They land via search or a link, take what they need, and
leave. They will not read your introduction.

That single difference drives everything else:

| | essay | technical document |
|---|---|---|
| reader | starts at the top | lands in the middle |
| reads | once, in order | repeatedly, in fragments |
| goal | be convinced | get unstuck |
| success | a persuaded reader | a reader who leaves quickly |
| structure serves | the argument | **the reader's need** |

So the question is never "what's my thesis?" It is **"what does the reader need, and which kind of
document serves that need?"**

There is no single tech-doc template. There is a small catalogue of **forms**, and picking the
right one is most of the job.

## 2. Diátaxis — the framework

The widely-adopted answer is **Diátaxis**, by Daniele Procida (<https://diataxis.fr>), used by
Django, Canonical, Cloudflare and many others. It says documentation divides into exactly **four
forms**, because reader needs divide along exactly two axes.

**The two axes, in the framework's own words:**

- **Action / cognition** — "action (practical knowledge, knowing *how*)" versus "cognition
  (theoretical knowledge, knowing *that*)".
- **Acquisition / application** — whether the reader is **acquiring** their craft or **applying** it.

Two axes, four quadrants, four forms:

```
                    ACQUISITION                    APPLICATION
                 (learning the craft)         (using the craft)
              ┌──────────────────────────┬──────────────────────────┐
   ACTION     │      TUTORIAL            │     HOW-TO GUIDE         │
  (knowing    │   learning-oriented      │   problem-oriented       │
    how)      │                          │                          │
              │  "Teach me by doing."    │  "I have a job to do."   │
              │  A lesson. You lead.     │  A recipe. They lead.    │
              ├──────────────────────────┼──────────────────────────┤
  COGNITION   │     EXPLANATION          │    TECHNICAL REFERENCE   │
  (knowing    │  understanding-oriented  │  information-oriented    │
    that)     │                          │                          │
              │  "Help me understand."   │  "Tell me the facts."    │
              │  Discussion. Why.        │  A map. Dry. Complete.   │
              └──────────────────────────┴──────────────────────────┘
```

| form | orientation | the reader is saying | shape |
|---|---|---|---|
| **Tutorial** | learning-oriented | *"I'm new — teach me by doing something that works."* | A guided lesson with a guaranteed outcome. You choose the path. |
| **How-to guide** | problem-oriented | *"I know roughly what I'm doing. Give me the steps."* | A recipe for one real task. Assumes competence. |
| **Technical reference** | information-oriented | *"What are the exact parameters?"* | Dry, complete, consistently structured. Describes the machine. |
| **Explanation** | understanding-oriented | *"Why is it built this way?"* | Discursive. Context, alternatives, trade-offs, history. |

### 2.1 The one rule that matters most

**Do not mix two forms in one document.** This is the single biggest cause of documentation that
feels bad to read, and it is almost always what has gone wrong.

- A **tutorial** that stops to explain theory loses the beginner, who needed momentum and a working
  result.
- A **reference** that tells a story becomes unsearchable — the reader wanted a table.
- An **explanation** stuffed with step-by-step commands buries the idea the reader came for.
- A **how-to** that teaches fundamentals wastes the time of someone who already knows them.

When a document feels wrong and you cannot say why, check whether it is trying to be two things.
The fix is a split plus a link, nearly every time.

## 3. The fifth form Diátaxis does not cover — the decision record

Diátaxis describes documentation *of a system*. It does not cover documenting **a choice made by
people at a point in time**, which is the thing future-you most reliably needs and least reliably
remembers.

That is the **ADR — Architecture Decision Record** (Michael Nygard's format). Short, dated, one per
decision, never edited after the fact — superseded rather than rewritten, so the record of what was
believed at the time survives.

Five lines is a legitimate ADR:

```markdown
# ADR-003: Use Git LFS for binary assets
Date: 2026-09-14
Status: Accepted
Context: Git stores compressed binaries as full copies; measured 6.7x repo bloat vs text.
Decision: Track .uasset/.umap/textures/audio via LFS from commit #1.
Consequences: Needs git-lfs on every clone; hosted quota applies. Rules must precede first commit.
Alternatives considered: plain git (repo bloat); Perforce (server overhead, team-scale tool).
```

In this vault, [[Decisions]] is the ADR log.

## 4. The forms this project actually uses

Phoenix is a learning project, so all five appear. Reach for them like this:

| when you… | write a… | goes in |
|---|---|---|
| study a tool/technique and want the facts recorded | **Research note** (an Explanation) | `03-research/` |
| plan milestones, risks and schedule | **Production plan** | `04-production/` |
| design what a feature is for the player | **Feature game-design spec** | `05-game-design/<Feature>.md` |
| design how a feature is built | **Feature technical spec** (applies [[OOP-Foundations]] §8) | `01-architecture/<Feature>-Tech.md` |
| add a network message | **Row in the protocol catalog** (Reference) | `01-architecture/Network-Protocol.md` |
| plan the next build step before coding | **Build-step spec** (Tutorial + acceptance criteria) | `02-build-steps/` |
| decide something you might second-guess later | **ADR entry** | `00-orientation/Decisions.md` |
| establish a rule for how code is written | **Reference** | `01-architecture/` |
| finish a work session | **Build log entry** | `00-orientation/Build-Log.md` |

## 5. Templates

### 5.1 Research note (Explanation) — `03-research/<Topic>-Research-<date>.md`

```markdown
---
title: <Topic> — Research
date: YYYY-MM-DD
status: research-draft
tags: [phoenix, research, <area>]
---

# <Topic> — Research

> [!info] **Scope.** What question this answers. **Facts only — no design decisions.**

## Revision history
| rev | date | what changed |

## Evidence tiers
[measured] > [source] > [doc] > [NOT verified]

## 1. The problem, stated concretely
## 2. Mechanism — how it actually works
## 3. Consequences
## 4. What to do / not do
## 5. Failure modes
## 6. NOT verified          ← never omit; this section IS the job
## Related
```

**The NOT-verified section is not an admission of sloppiness — it is the most valuable part.** It
marks the boundary of what you actually established, so a later reader knows which claims to lean
on and which to re-check.

### 5.2 Build-step spec (Tutorial) — `02-build-steps/Step-N-<name>.md`

```markdown
## Goal                    ← one paragraph, what works at the end
## Explicitly out of scope ← the fence. Resist adding things.
## Prerequisites
## The work                ← classes, code, in build order
## Acceptance criteria     ← [ ] checkboxes, each objectively true or false
## Learning checklist      ← what you should be able to EXPLAIN afterwards
```

The last two sections are what make it a tutorial rather than a to-do list. Acceptance criteria
give a guaranteed outcome; the learning checklist names the actual point of the exercise.
[[Step-1-Move-and-Attack]] already follows this.

### 5.3 Reference — `01-architecture/*.md`

```markdown
## <Thing>
| field | type | meaning | default |
```

Dry. Complete. Consistent. No narrative, no persuasion, no history. If you find yourself explaining
*why*, that paragraph belongs in an Explanation — link to it.

### 5.4 Build log entry

```markdown
## YYYY-MM-DD — <one-line summary>
- What got built / changed
- What broke and why
### Next up
- [ ] ...
```

Newest at the top. Append-only; do not retro-edit — a wrong belief you held on a Tuesday is useful
data.

## 6. Universal rules, regardless of form

1. **Lead with the conclusion.** The reader landed mid-document; do not make them read an
   investigation narrative to reach the answer. State the finding, then support it.
2. **Show the artifact, don't describe it.** Quote real code with `file:line`, paste the real
   output, show the real number. *"The filter is applied in the service"* is unverifiable prose;
   six quoted lines are checkable.
3. **Every claim carries how it was established.** Measured > read in the source > official docs >
   someone said so. A note where all claims look equally certain is a note that cannot be trusted.
4. **Say what you did not verify.** Explicitly, in its own section.
5. **Use the source's own vocabulary.** If Unreal calls it a `SpringArmComponent`, call it that —
   never a tidier invented synonym. The reader will search upstream for your word and must find it.
6. **A reference the reader cannot follow is a claim they cannot check.** Link `[[Note#Section]]`,
   never a bare "see §4" pointing into a different note.
7. **Date everything, and supersede rather than silently rewrite.** A struck-through wrong claim
   with its correction teaches more than a clean page that hides the change.

## 7. How the existing notes map

The vault already followed this before it had a name:

| note | form | orientation |
|---|---|---|
| [[HOME]] | — | map of content / hub |
| [[Design-Doc]] | Explanation | vision, pillars, scope |
| [[Decisions]] | ADR log | decisions + rationale, dated |
| [[Build-Log]] | — | chronological record |
| [[Architecture]] | Explanation | where code goes, and why |
| [[Coding-Conventions]] | **Reference** | rules, lookup-shaped |
| [[System-Design]] | **Reference** | per-system data schemas |
| [[Step-1-Move-and-Attack]] | **Tutorial** | guided lesson + acceptance criteria |
| [[Git-LFS-Research-2026-09-14]] | Explanation | understanding-oriented |

**The gap: no How-to guides yet.** That is correct for now — how-tos are for a competent user with
a recurring task, and the recurring tasks do not exist yet. Expect the first ones around build step
3–4: *"How to add a new enemy type"*, *"How to regenerate compile_commands.json"*.

## 8. Glossary

Per house rule, external terminology anchored to the source that settled it.

| term | source | checked |
|---|---|---|
| Diátaxis; Tutorial / How-to guide / Technical reference / Explanation | <https://diataxis.fr> | 2026-09-14 |
| learning- / problem- / information- / understanding-oriented | <https://diataxis.fr> | 2026-09-14 |
| action vs cognition; acquisition vs application | <https://diataxis.fr/foundations/> | 2026-09-14 |
| ADR (Architecture Decision Record) | Michael Nygard's format | 2026-09-14 — **[NOT verified]** against the original article; format reproduced from house convention |

## Related
- [[HOME]] — vault index
- [[Git-LFS-Research-2026-09-14]] — the first note written to this framework
