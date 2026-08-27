# 📐 codent — SPEC-DRIVEN DEVELOPMENT (SDD) PROCESS

**Version:** 1.0
**Status:** Active. Mandatory for all feature work in this repository.
**Audience:** The LLM agent (Claude) and the humans (royo, surya) it pairs with.

> **One-line summary.** No feature gets coded until a unit-decomposed spec with per-unit tests and a
> wiring plan exists, is approved, and the LLM has passed a Goldfish self-comprehension check
> against it. Then the LLM builds **one unit at a time** — develop → test → run → tracker update →
> summary block — and only then moves on.

This document is the operating manual the LLM reads whenever a developer says *"build feature X"*.
It synthesises two sources the team has agreed to follow:

1. **Dave Rensin — *Elephants, Goldfish and the New Golden Age of Software Engineering***. The
   "design is the new code" principle, the Elephant-Goldfish Model, and the Goldfish comprehension
   test.
2. **Kiro's Spec-Driven Development.** Discrete tasks that map back to requirements (EARS notation),
   executable specs, steering documents.

`BUILD_SPEC.md` is the **architectural source-of-truth** for codent itself (vision, pipeline, stack,
data contracts, risk formula, design decisions). **This SDD process governs every spec that lands
*on top* of that architecture.**

---

## 📌 TABLE OF CONTENTS

1. [Mental Model](#1-mental-model)
2. [Roles & Ownership](#2-roles--ownership)
3. [Repository Layout](#3-repository-layout)
4. [End-to-End Flow](#4-end-to-end-flow)
5. [Spec Authoring Mode](#5-spec-authoring-mode)
6. [The Goldfish Gate](#6-the-goldfish-gate)
7. [Build Mode — The Per-Unit Loop](#7-build-mode--the-per-unit-loop)
8. [Wiring & Wiring Test](#8-wiring--wiring-test)
9. [Multi-Session Continuation](#9-multi-session-continuation)
10. [Tracker Schema](#10-tracker-schema)
11. [Status Vocabulary](#11-status-vocabulary)
12. [Test Stack & Conventions](#12-test-stack--conventions)
13. [Relationship to BUILD_SPEC.md](#13-relationship-to-build_specmd)
14. [Anti-Patterns the LLM Must Refuse](#14-anti-patterns-the-llm-must-refuse)
15. [Quick Reference — Developer Prompt Templates](#15-quick-reference--developer-prompt-templates)
16. [Multi-Developer Workflow (Git)](#16-multi-developer-workflow-git)

---

## 1. Mental Model

> *"If we lean entirely on AI to write code without forcing those design judgments into
> human-readable, rigorously tested documents, we will lose the ability to confidently put our names
> on the work we are supposed to stand behind."* — Rensin

The symptom this process exists to prevent: a developer can describe *what* the app does but not
*how* any given piece works, because the LLM wrote it in a fugue state and nobody reconstructed the
reasoning.

Two non-negotiable beliefs encoded throughout this doc:

- **The spec is the source of truth, not the code.** Code is the compiled output of a spec. If the
  spec is missing, ambiguous, or out of date, we stop coding and fix the spec.
- **A future fresh LLM session ("Goldfish") must be able to reconstruct intent from the spec alone.**
  If a brand-new Claude instance — with no memory of our chat — reads the spec and cannot say what
  the feature does and how it works, the spec is broken. Before any code lands, we test the spec
  against a Goldfish (see §6).

### The two modes the LLM operates in

The LLM is **either authoring a spec or building from a spec, never both at once.**

| Mode | Trigger | Output | Code allowed? |
|---|---|---|---|
| **Spec Authoring** | Developer hands the LLM a feature ask + this doc | A filled-out `spec.md` under `specs/<owner>/<feature-slug>/` plus tracker rows | **NO.** Pseudocode and file paths only. |
| **Build** | Developer says "build the next unit" against an existing approved spec | Real code + tests + tracker updates + a unit summary block | **YES**, but strictly within the unit boundaries the spec declares. |

If the developer asks the LLM to build something without a spec, the LLM **refuses and offers to
author one**. This is not optional.

---

## 2. Roles & Ownership

### Humans

- **royo** — owns specs under `specs/royo/`
- **surya** — owns specs under `specs/surya/`

Ownership means: the spec was scoped, reviewed, and approved by that person. They are the one who
reviews the LLM's unit summaries and signs off "ready to ship". They may pair with the LLM across
multiple sessions on the same spec; the tracker carries state between sessions so the work survives
context loss.

A spec is allocated to an owner at authoring time. If ownership transfers, the spec folder is
*moved* (not copied) and the tracker rows have their `owner` column updated.

### The LLM

The LLM is the executor in both modes. It is **not** a co-owner. It does not decide what to build,
what to defer, what to ship. It interrogates, decomposes, builds, tests, and reports. The owning
human exercises judgment; the LLM exercises rigor.

---

## 3. Repository Layout

Everything SDD-related lives under the project root.

```
/Users/royo/Resume_projects/codent/
├── BUILD_SPEC.md              # Architectural source-of-truth
├── SDD_PROCESS.md             # This file — the methodology
├── ARCHITECTURE_PROPOSAL.md   # Historical: the draft BUILD_SPEC.md resolved
└── specs/
    ├── _template.md           # The per-feature spec template
    ├── _tracker.csv           # Cross-spec unit-level progress tracker (THE source of "where are we")
    ├── royo/
    │   ├── README.md          # royo's queue + how to prompt the LLM
    │   └── <feature-slug>/    # One folder per feature royo owns
    │       ├── spec.md        # The contract (mostly immutable after approval)
    │       └── BUILD_LOG.md   # Append-only narrative of what was built each session
    └── surya/
        ├── README.md
        └── <feature-slug>/
            ├── spec.md
            └── BUILD_LOG.md
```

**Per-spec files explained:**

- `spec.md` — the contract. Authored once, approved by the owner, then treated as immutable for the
  duration of the build. Edits to an approved spec require a numbered revision note appended at the
  bottom of the file *and* a tracker note. Do not silently rewrite an approved spec.
- `BUILD_LOG.md` — the story. Appended to at the end of each build session. Each entry includes the
  unit summary blocks emitted that session, any blockers, and a "where to resume next" pointer.
- `specs/_tracker.csv` — the index. One row per unit, across all specs and all owners. This is the
  file the LLM reads first when asked "what's next?".

Tests do **not** live in `specs/`. They live in `tests/unit/` and `tests/integration/` — but the
spec declares the test file paths up front so the LLM doesn't drift.

---

## 4. End-to-End Flow

A feature moves through these stages, in order. No skipping.

```
┌───────────────────────────────────────────────────────────────────┐
│  STAGE 0 — DEVELOPER HANDS OFF THE FEATURE ASK                    │
│  Developer prompts LLM with: this doc + feature description       │
│  + owner (royo/surya). See §15 for the exact prompt template.     │
└─────────────────────────────┬─────────────────────────────────────┘
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│  STAGE 1 — SPEC AUTHORING (LLM, no code)                          │
│  LLM enters Spec Authoring Mode. Interrogates the developer.      │
│  Produces a filled spec.md from specs/_template.md. Decomposes    │
│  into units. Adds one tracker row per unit (status=spec_drafted). │
│  See §5.                                                          │
└─────────────────────────────┬─────────────────────────────────────┘
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│  STAGE 2 — GOLDFISH GATE (LLM, no code)                           │
│  LLM does a cold re-read self-check against the spec.             │
│  Reports any ambiguity. Developer reviews + approves.             │
│  On approval: tracker rows flip from spec_drafted → not_started.  │
│  See §6.                                                          │
└─────────────────────────────┬─────────────────────────────────────┘
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│  STAGE 3 — BUILD LOOP (LLM, code allowed)                         │
│  For each unit in declared order:                                 │
│    a. Read spec.md + tracker row for this unit                    │
│    b. Implement (only the files the spec lists)                   │
│    c. Write tests at the paths the spec declares                  │
│    d. Run tests                                                   │
│    e. Update tracker row (status, algo summary, test counts...)   │
│    f. Emit a unit summary block (§7.6) to the developer           │
│    g. Append a BUILD_LOG.md entry                                 │
│    h. STOP. Wait for the developer to say "next" before           │
│       moving on, unless the developer pre-authorized continuous   │
│       mode for this spec.                                         │
│  See §7.                                                          │
└─────────────────────────────┬─────────────────────────────────────┘
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│  STAGE 4 — WIRING & WIRING TEST                                   │
│  Treated as the final unit of every spec. Wire the unit-level     │
│  pieces into the assembled feature. Run the wiring test the spec  │
│  declared. Update tracker (wiring row → done). See §8.            │
└─────────────────────────────┬─────────────────────────────────────┘
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│  STAGE 5 — FINAL SUMMARY + INDEX UPDATE                           │
│  LLM emits a final spec-level summary, ensures all tracker rows   │
│  for this spec are status=done, appends a "feature complete"      │
│  marker to BUILD_LOG.md, and notes the entry in the Specs Index   │
│  at BUILD_SPEC.md §1.                                             │
└───────────────────────────────────────────────────────────────────┘
```

---

## 5. Spec Authoring Mode

**Iron rules for this mode:**

1. **No code.** Not even a one-line snippet outside `pseudocode` blocks. Resist the urge.
2. **Interrogate before authoring.** The LLM asks clarifying questions until the feature is
   concrete. If the developer says "you decide", the LLM proposes a default and the developer
   confirms.
3. **Follow `specs/_template.md` exactly.** Section order, headings, frontmatter — all match.
4. **Decompose into units only the LLM can build atomically.** A unit is "one thing that can be
   implemented and tested in isolation, in a single short build cycle, without needing other
   not-yet-built units to be present". Typical unit size: a single function/class/module plus its
   tests. If a unit can't be tested without another unit existing, merge them or reorder.
5. **Each unit declares its algorithm/logic in prose, not just its interface.** The spec must read
   like a thoughtful design memo, not an API stub. A reader should understand *why* the algorithm is
   what it is.
6. **Enumerate every file the unit will create or modify.** Path + brief role. This is the strongest
   guardrail against scope creep during build.
7. **Each unit declares its test cases up front** — inputs, expected outputs, the file path each
   test will live in. The tests are part of the contract, not a build-time afterthought.
8. **The Wiring Plan is mandatory**, even for a one-unit spec (in which case it's "no wiring needed;
   the single unit is the feature").
9. **The Wiring Test is mandatory** and exercises the assembled feature end-to-end — at the seam the
   next system upstream actually touches.

### EARS notation for requirements

Acceptance criteria are written in [EARS](https://alistairmavin.com/ears/) (Easy Approach to
Requirements Syntax). The five canonical forms:

- **Ubiquitous:** `The <system> shall <response>.`
- **Event-driven:** `When <trigger>, the <system> shall <response>.`
- **State-driven:** `While <state>, the <system> shall <response>.`
- **Optional feature:** `Where <feature is included>, the <system> shall <response>.`
- **Unwanted behaviour:** `If <trigger>, then the <system> shall <response>.`

Every requirement gets an ID like `R1`, `R2`, … and every unit in the Unit Decomposition section
cites the requirement IDs it satisfies. That mapping is what makes the spec "executable" in the Kiro
sense — no orphan requirements and no orphan units.

### What to challenge in the developer's ask (the Sycophant Challenge, baked in)

Before writing the spec, the LLM **must push back at least once** if any of these are true:

- The feature could be a config change instead of code → suggest that. (codent has a real config
  surface — `BUILD_SPEC.md §11`. A "make it less noisy" ask is a threshold change, not a feature.)
- The feature duplicates something the codebase already does → cite the file and ask if extending is
  preferable to building new.
- The proposed boundary is unclear → propose an alternative boundary.
- **The feature adds an LLM call to the per-PR path.** `BUILD_SPEC.md §4` caps that path at four
  calls, with a batching rule. Any spec that adds a fifth unconditional call, or that fans a batched
  call out per-file or per-finding, must amend §4 *first* — flag this loudly and stop.
- **The feature reaches around an interface.** `BUILD_SPEC.md §6` requires the pipeline to depend on
  `LLMProvider` / `SandboxRunner` / `Embedder` / `Linter`, never on a concrete implementation. A
  spec whose `Files touched` puts a concrete class inside `pipeline/` is violating the architecture.
- **The feature weakens the sandbox boundary** (`BUILD_SPEC.md §10`) or lets `.codent.yml` control
  anything beyond numbers and path globs (`§11`). Both are security boundaries; flag and ask.

> *Pro-tip:* asking "Why do you think that?" is the LLM's best friend for snapping the developer out
> of a half-thought-through idea. Use it.

---

## 6. The Goldfish Gate

After the draft spec is written, **the LLM does a cold-read self-check before handing the spec back
to the developer**. This is the Goldfish Protocol, performed in-session by the same LLM but with the
explicit framing "I am reading this for the first time".

The check has three passes:

1. **Comprehension pass** — "If I had not just written this, could I tell the developer what this
   feature does and how it works from the spec alone?" If no → add the missing context, repeat.
2. **Critic pass** — "Adopt the role of an expert technical reviewer. What did I miss? What edge
   cases are unhandled? What assumptions are unstated?" Update the spec until critiques fall into
   the "nit-pick" category.
3. **Implementation-readiness pass** — "If I started building from this spec right now, are there
   any decisions I would still have to invent on the fly?" If yes → push those decisions back into
   the spec.

The LLM reports the result of each pass in plain English ("here's what I changed during Goldfish
review", "here are the items I want your call on before approving") and **only then** asks the
developer to approve.

On approval:

- Every tracker row for this spec flips `status: spec_drafted → not_started`.
- Owner appends an "approved on YYYY-MM-DD" line at the top of the spec's BUILD_LOG.md.
- The spec is now treated as immutable for the duration of the build.

---

## 7. Build Mode — The Per-Unit Loop

### 7.1 Mode entry

The LLM enters Build Mode only when:

- An approved `spec.md` exists at the indicated path, AND
- At least one tracker row for that spec has status `not_started` or `in_progress`.

If neither is true, the LLM refuses and either (a) asks the developer to approve an existing draft,
or (b) suggests authoring a new spec.

### 7.2 Locate the next unit

The LLM reads `specs/_tracker.csv`, filters to rows where `spec_id` matches
`<owner>/<feature-slug>`, sorts by `unit_id`, and picks the **first row with status `in_progress`
(a partially-finished resume) or, if none, the first row with status `not_started`**.

### 7.3 Re-read the spec for this unit only

Open `spec.md`, scroll to the matching unit. Re-read: Purpose, Inputs/Outputs, Algorithm/Logic,
Files touched (the exact list), Test cases (the exact list, with paths), Acceptance criteria.

If anything is unclear *at this point*, stop. Ask the developer. Do not improvise.

### 7.4 Implement

Hard rules:

- **Only touch files the unit's `Files touched` list names.** If you genuinely need to touch another
  file, stop and ask. Adding "one more file" is how vibe-coding sneaks back in.
- **No new abstractions** beyond what the spec describes. No "while I was here I refactored…".
  Refactors require their own spec.
- **No new dependencies** without a tracker note. If a unit needs a new library, that should have
  been listed in the spec; if it wasn't, stop and update the spec first.
- **Match the project style** (`BUILD_SPEC.md §5`, `§6`): Python 3.11, FastAPI, Pydantic models for
  every data contract, `async def stage(state, deps) -> state` for pipeline stages, and dependency
  injection through `Deps` rather than module-level singletons.
- **Respect the architectural invariants** in `BUILD_SPEC.md`: the LLM call budget (§4), the
  interface rule (§6), the line-number rule (§7), the sandbox posture (§10), and the config
  restrictions (§11). If a unit as written would break one of these, that is a spec bug — stop and
  raise it (§9.5), don't code around it.

### 7.5 Test

- Write the test files at the paths the spec declared.
- Use **pytest** (see §12).
- Each test case from the spec must map to at least one assertion. Extra tests are fine; missing
  tests are not.
- Run the tests. Report pass/fail counts honestly.
- **A failing test does not mean "move on and fix later".** If a test fails, the unit is
  `in_progress` (or `blocked` if the cause is external) — never `built` and never `done`.
  Investigate, fix, re-run.

### 7.6 Emit the Unit Summary Block

After tests pass, before touching the next unit, the LLM prints **exactly this block** as its primary
user-facing output. This is the artefact the developer reads to verify the work without diffing the
code:

```
================ UNIT COMPLETE ================
Spec ID         : <owner>/<feature-slug>
Unit            : <unit_id> — <unit_name>
Status          : built | tested | wired | done | blocked
Algorithm       : <1–3 sentences describing the actual algorithm/
                  logic that was implemented, in plain English.
                  This is the line that defeats vibe-coding —
                  the developer should be able to explain the
                  feature back from this alone.>
Files touched   :
  - <path>  (new | modified, +N/-M lines)
  - ...
Tests added     : <count>  at  <test file path>
Tests run       : <N passed / M failed / K skipped>
Tracker updated : yes
BUILD_LOG.md    : appended
Next unit       : <next unit_id>   |   WIRING   |   FEATURE COMPLETE
===============================================
```

Mandatory; no exceptions. If a field is N/A, write "N/A" — don't omit the line.

### 7.7 Update the tracker

Open `specs/_tracker.csv`, find this unit's row, fill in:

- `status` → `built` (tests pass in isolation), `done` (only after the wiring step closes everything
  out, OR if this unit is genuinely standalone with no wiring required)
- `algorithm_summary` → the same 1–3 sentence prose summary that went into the unit block
- `files_touched` → comma-separated list (double-quote the CSV cell to escape the commas)
- `tests_added` → count of new test functions/cases
- `tests_passed` / `tests_failed` → counts from the run
- `date_completed` → ISO date (YYYY-MM-DD)
- `session_id` → an opaque string per build session — see §9.4
- `notes` → anything important the next session must know

### 7.8 Append to BUILD_LOG.md

A short prose entry per unit, dated, including the unit block above and any context that won't fit
in the tracker. When a teammate joins later they should be able to read BUILD_LOG.md top-to-bottom
and understand how the feature was built.

### 7.9 Stop. Wait.

After emitting the unit block and updating the tracker / BUILD_LOG, **the LLM does not auto-advance
to the next unit by default**. It waits for the developer to say "next" (or some variant).

**Exception:** the developer may pre-authorise continuous build with a phrase like *"build all units
in this spec without stopping"* — in which case the LLM proceeds unit-by-unit, still emitting a
summary block per unit and still updating the tracker after each, but does not pause for
confirmation. The developer can interrupt at any time.

This pause is non-negotiable for the *first* unit of a brand-new spec, even under continuous mode —
that way the developer sees the LLM's interpretation early.

---

## 8. Wiring & Wiring Test

After the last functional unit, one final unit always exists: the **Wiring Unit**. The spec lists it
explicitly (usually as `wiring`). Its job is to:

- Compose the previously-built units into the assembled feature exactly per the spec's `Wiring Plan`.
- Touch only the integration files the spec lists (e.g. a stage registration in
  `src/codent/pipeline/runner.py`, a route registration in `src/codent/main.py`, a dependency wired
  into `Deps`).
- Execute the **Wiring Test** declared in the spec. This is the end-to-end test that verifies the
  whole feature works at the seam a real caller would use (an HTTP request to the webhook endpoint,
  a full `run_pipeline(state)` invocation with fake `Deps`, etc.). It does **not** mock the inner
  units — that's the point.

The Wiring Unit gets the same unit-block treatment as any other unit. Its `status` ends at `done`,
and when it is marked `done`, the LLM flips every prior unit of the same spec from `built` → `done`.

If the wiring test fails, the LLM **does not** edit the previously-built units to make the test pass
without checking with the developer. A wiring-test failure usually means a wiring-plan bug or a
missed assumption — both are spec issues, not code issues.

---

## 9. Multi-Session Continuation

### 9.1 The invariant that makes this work

Every piece of state the next session needs lives in **one of three files** — all under git:

| File | What it holds | Who writes it |
|---|---|---|
| `spec.md` | The contract. Doesn't change once approved. | Authoring session |
| `specs/_tracker.csv` | The unit-level status across all specs. | Every build session |
| `<spec>/BUILD_LOG.md` | The narrative + per-session "resume here" pointer. | Every build session |

If a session ends mid-unit, the LLM updates that unit's tracker row to `status=in_progress` (not
`built`) with a note like `partial: implemented foo() but not bar() — see BUILD_LOG.md last entry`,
appends a matching BUILD_LOG entry, and that's enough. The next session can fully reconstruct where
to resume.

### 9.2 Developer prompt — starting a brand-new spec session

> *"Read **BUILD_SPEC.md** and **SDD_PROCESS.md**. I want to add a new feature owned by
> **<royo | surya>**. Here's the feature description: <one paragraph or bullet list>. Enter **Spec
> Authoring Mode** and produce a draft spec under `specs/<owner>/<your-suggested-slug>/spec.md`,
> plus a tracker entry per unit. Run the **Goldfish Gate** before handing it back. Do not write any
> code."*

### 9.3 Developer prompt — continuing an existing spec across sessions

> *"Read **SDD_PROCESS.md** and `specs/<owner>/<feature-slug>/spec.md`. Open `specs/_tracker.csv`
> and find the next unit for `<owner>/<feature-slug>` that is `not_started` or `in_progress`. Read
> `specs/<owner>/<feature-slug>/BUILD_LOG.md` for any handoff notes from the previous session. Then
> **enter Build Mode** and proceed with that unit. Emit a unit summary block when done and stop."*

Or — if the developer wants the LLM to plow through:

> *"... Same as above, but build all remaining units in this spec without stopping between them,
> until either the spec is complete or a test fails."*

### 9.4 Session IDs

A "session" is one continuous chat session with the LLM. The LLM mints a `session_id` at the start
of a build session — a short kebab-case string like `2026-08-27-royo-pm` (date + owner + am/pm),
unless the developer provides one. It writes this `session_id` into every tracker row it updates that
session and references it in BUILD_LOG entries.

### 9.5 Resuming after the spec itself needs to change

Sometimes mid-build, the spec turns out to be wrong (an assumption broke, a missing dependency
surfaced, the developer changed their mind). The procedure:

1. **Stop building.** Do not silently fix it in code.
2. **Append a numbered "Revision" block at the bottom of `spec.md`** describing what changed and
   why. Do not edit the body of the approved spec — append.
3. **Re-run the Goldfish Gate** (§6), narrowly scoped to the affected units only.
4. **Have the owner re-approve.** Tracker rows for affected units flip back to `not_started` if
   their inputs changed materially, or stay `built` if the revision is purely additive.
5. **Then resume.**

**If the revision touches an architectural invariant** — the LLM call budget, an interface boundary,
the risk formula's weights, the sandbox posture, or the config restrictions — then `BUILD_SPEC.md`
needs a revision block too (`BUILD_SPEC.md §14`), and that revision lands **before** the spec's does.
The architecture doc is upstream of every spec; it never trails one.

This is friction by design. It's faster than the alternative (silent drift → re-debug → throw it all
away).

### 9.6 Auto-discovery & resume rules ("just continue my work")

The strongest form of multi-session continuation is the one where the developer doesn't have to
remember which spec they were on. The LLM does the lookup itself against `specs/_tracker.csv`,
filtered by the developer's `owner` column.

When the developer's prompt says *"find my next unit"* (or any equivalent — see §15.G), the LLM
applies this precedence:

| Step | Condition | Action |
|---|---|---|
| 1 | Any row has `status: in_progress` | Resume **that** unit. *Iron rule: never start new work while a partial sits unfinished.* |
| 2 | A spec has at least one `done`/`built` row **and** at least one `not_started` row | Build the next `not_started` unit in that spec, by `unit_id` order. |
| 3 | A spec has only `not_started` rows (newly approved, not yet started) | Build `unit-01` of that spec. |
| 4 | No remaining work for this owner | Report: *"you're caught up, no pending units for `<owner>`"*. |

**The ambiguity rule.** If steps 1, 2, or 3 produce **more than one candidate spec**, the LLM **must
stop and ask the developer which one**, listing each candidate with a one-line progress summary:

> *"You have two specs in flight: `royo/prefilter-gate` — 3/5 units done; `royo/ast-facts-block` —
> 1/4 units done. Which one?"*

Silent picks in ambiguous cases are forbidden — that's the failure mode where the AI sprints off a
cliff because it wants to be helpful. Asking is cheap; restoring lost work is not.

**The Goldfish-on-resume rule.** Even on auto-resume, **before any code is written**, the LLM must
read the spec's `spec.md` and `BUILD_LOG.md` and **state in 2–3 sentences what the unit is about**.
This is the developer's instant sanity check that the LLM picked up the right thread.

**Cost.** `_tracker.csv` is small (one row per unit, across all specs and owners). Read the whole
file every session — don't build a "what's next" cache or an index file. The CSV *is* the index.

---

## 10. Tracker Schema

`specs/_tracker.csv` is a flat CSV with this header row:

```
spec_id,owner,feature,unit_id,unit_name,status,algorithm_summary,files_touched,tests_added,tests_passed,tests_failed,date_completed,session_id,notes
```

| Column | Type | Purpose |
|---|---|---|
| `spec_id` | string | `<owner>/<feature-slug>` — the spec this row belongs to. E.g. `royo/prefilter-gate`. |
| `owner` | enum | `royo` or `surya`. |
| `feature` | string | Human-readable feature title. Duplicated across all rows of the same spec for easy filtering. |
| `unit_id` | string | E.g. `unit-01`, `unit-02`, …, `wiring`. Ordered, zero-padded. |
| `unit_name` | string | Short imperative title. E.g. "Compute per-file lint findings". |
| `status` | enum | See §11. |
| `algorithm_summary` | string | 1–3 sentences in plain English. Filled at `built` / `done` time. The line that defeats vibe-coding. |
| `files_touched` | string | Comma-separated paths. Quote the cell so the CSV stays valid. |
| `tests_added` | int | Count of new test functions for this unit. |
| `tests_passed` | int | From the most recent run. |
| `tests_failed` | int | From the most recent run. Zero is the only acceptable value at `done`. |
| `date_completed` | ISO date | YYYY-MM-DD, set when the row flips to `built`/`done`. Empty otherwise. |
| `session_id` | string | The session in which the row was last meaningfully updated. |
| `notes` | string | Free-form. Anything the next session must know. Keep short — long notes belong in BUILD_LOG.md. |

CSV editing rules:

- Quote any cell containing a comma or newline.
- Append-only at the row level — a unit's row stays in place once written; we *update* it in place
  rather than adding new rows. The history is in BUILD_LOG.md.
- The LLM may use Python's `csv` module or simple `Edit` tool replacements — either is fine, as long
  as the file remains valid CSV after the operation.

---

## 11. Status Vocabulary

A unit row's `status` value moves through this state machine:

```
   spec_drafted        ← row exists, spec not yet approved
        │ (Goldfish Gate passes + owner approves)
        ▼
   not_started         ← ready to build
        │
        ▼
   in_progress         ← partial build, will resume
        │
        ▼
   built               ← unit's own code + tests pass in isolation
        │ (the wiring unit completes successfully)
        ▼
   done                ← part of a shipped, wired feature

   ── orthogonal ──
   blocked             ← can't proceed; reason in notes
```

- `built` is the typical end-state for a non-wiring unit *during* a build cycle.
- `done` is reserved for after the wiring unit has confirmed the assembly works. When the wiring unit
  closes, the LLM flips every other unit of the same spec from `built` → `done`.
- `blocked` is always paired with a `notes` value explaining what's blocking and what unblocks it.

The LLM never invents new status values; if a new state is genuinely needed, propose it via a
revision to this doc.

---

## 12. Test Stack & Conventions

codent is a single Python service. There is no frontend, and therefore no second test stack.

| Layer | Framework | Location pattern |
|---|---|---|
| Unit | **pytest** (+ `pytest-asyncio` for `async def` stages) | `tests/unit/test_<unit>.py` |
| Wiring / integration | **pytest** + `httpx.AsyncClient` against the FastAPI app | `tests/integration/test_<feature>_wiring.py` |
| Fixtures | Sample diffs, recorded webhook payloads, tiny sample repos | `tests/fixtures/` |

Run commands the spec should reference verbatim:

```bash
# a single unit
pytest tests/unit/test_<unit>.py -v

# the wiring test for a feature
pytest tests/integration/test_<feature>_wiring.py -v

# everything
pytest -q
```

### codent-specific testing rules

These apply to every spec unless the spec explicitly overrides them with a reason:

1. **No real LLM calls in tests.** Ever. Every test injects a fake `LLMProvider` through `Deps` and
   asserts on the prompt it was handed and the structured object it returned. A test that would
   spend money or need network is not a unit test.
2. **No real GitHub calls in tests.** Same rule — fake the client, assert on the calls.
3. **No real sandbox execution in unit tests.** Inject a fake `SandboxRunner` returning a canned
   `SandboxResult`. The *real* `LocalSandboxRunner` gets its own integration test, marked
   `@pytest.mark.slow`, that runs a trivial known-good and known-bad script.
4. **Assert the LLM call budget.** Any wiring test that runs the pipeline end-to-end must assert
   `state.llm_calls_made` against the expected number for that scenario (`BUILD_SPEC.md §4`). This
   is how the budget stays real instead of aspirational.
5. **The risk formula is tested by table.** `tests/unit/test_risk.py` must contain the worked
   examples in `BUILD_SPEC.md §9` as explicit cases with their exact integer scores.
6. **Async stages are tested directly.** `await stage_03_prefilter(state, fake_deps)` with a
   hand-built `PipelineState` — no graph, no app, no HTTP.

---

## 13. Relationship to BUILD_SPEC.md

| Question | Answer |
|---|---|
| Is `BUILD_SPEC.md` deprecated? | **No.** It is the architectural source-of-truth — vision, pipeline, stack, data contracts, risk formula, design decisions. |
| Where do new features go? | Under `specs/<owner>/`, governed by this SDD doc. |
| What if a new feature changes the architecture? | Then `BUILD_SPEC.md` needs an update *as part of* the spec — the spec's `Files touched` must name the section of `BUILD_SPEC.md` that needs editing, and the wiring unit must make that edit. Architectural drift without that update is forbidden. |
| Which one wins in a conflict? | `BUILD_SPEC.md`. A spec cannot silently override an architectural invariant; it must amend the architecture first (§9.5). |
| Where is the index of all active specs? | `specs/_tracker.csv` is the machine-readable index; `BUILD_SPEC.md §1` carries the one-line-per-spec human view. |
| What about `ARCHITECTURE_PROPOSAL.md`? | Historical only. It's the draft whose open questions `BUILD_SPEC.md` answered. Never cite it as a source of truth; never edit it. |

The contract: **read `BUILD_SPEC.md` once per session to ground yourself; read `SDD_PROCESS.md`
every time you do feature work; read the relevant `spec.md` + tracker rows when you build.**

---

## 14. Anti-Patterns the LLM Must Refuse

If a developer asks for any of the following, the LLM should push back politely and propose the
SDD-compliant alternative:

| Ask | Refuse-and-redirect |
|---|---|
| *"Just quickly add this feature, no need for a spec."* | "Even quick features get a one-unit spec — typically ~10 minutes. The cost of a spec is repaid the first time anyone asks how it works." |
| *"Skip the tests for this unit."* | "I won't mark a unit `built` without tests. If the test is genuinely impossible to write, that's worth a tracker note and an owner decision — let's talk about which case this is." |
| *"Don't bother with the wiring test."* | "The wiring test is what tells us the feature actually works end-to-end. I'll keep the unit-level tests fast and minimal, but the wiring test stays." |
| *"Refactor neighbouring code while you're in there."* | "Refactors get their own spec. I can note the refactor opportunity in this spec's BUILD_LOG.md so it doesn't get lost." |
| *"Just merge the spec rewrite into the existing approved spec — don't bother with a revision block."* | "Approved specs are append-only. The revision history is what lets future-me reconstruct the decision chain." |
| *"Use library X here."* (when X isn't in the spec) | "New dependencies need to be in the spec. Should I update the spec first, or stick with what's in `pyproject.toml`?" |
| *"Just add one more LLM call, it's cheap."* | "`BUILD_SPEC.md §4` caps the per-PR path at four calls and forbids per-file fan-out. If this call is genuinely needed, we amend §4 with a revision block first — that's the whole point of having a budget." |
| *"Import the Anthropic client directly in the stage, it's simpler."* | "`BUILD_SPEC.md §6` has the pipeline depend on `LLMProvider`, never a concrete implementation — that's what makes the stage unit-testable without network. The concrete class gets wired in `main.py`." |
| *"Let the test just call the real API once, to be sure."* | "§12 rule 1: no real LLM calls in tests. I'll add a fake provider that returns the exact structured object, and assert on the prompt we built." |
| *"Loosen the sandbox so the tests can install packages."* | "That's a §10 security boundary. If generated tests need dependencies, that's a spec question about how we prepare the checkout — not something we solve by widening the sandbox." |

---

## 15. Quick Reference — Developer Prompt Templates

Copy/paste these to drive an LLM session.

### A. Start a new feature spec

```
Read BUILD_SPEC.md and SDD_PROCESS.md.
Owner: <royo | surya>
Feature: <one paragraph describing what and why>

Enter Spec Authoring Mode. Interrogate me on anything ambiguous,
then produce a draft spec at specs/<owner>/<your-suggested-slug>/spec.md
and add unit rows to specs/_tracker.csv. Run the Goldfish Gate before
handing it back. No code yet.
```

### B. Approve a draft spec and start building

```
The draft at specs/<owner>/<feature-slug>/spec.md is approved.
Flip the tracker rows for this spec from spec_drafted to not_started,
append the approval line to BUILD_LOG.md, then enter Build Mode for
the first unit. Stop after the first unit and emit the unit summary
block.
```

### C. Continue a spec across sessions (resume)

```
Read SDD_PROCESS.md and specs/<owner>/<feature-slug>/spec.md.
Open specs/_tracker.csv and the spec's BUILD_LOG.md. Find the next
unit for <owner>/<feature-slug> that is in_progress or not_started.
Enter Build Mode. Build that one unit, run its tests, update the
tracker, emit the unit summary block, append BUILD_LOG.md, and stop.
```

### D. Run continuously to completion

```
Read SDD_PROCESS.md and specs/<owner>/<feature-slug>/spec.md.
Build all remaining units in this spec without stopping between
them, including wiring and wiring test, until either the spec is
complete or a test fails. Emit a unit summary block after each
unit. Update the tracker and BUILD_LOG.md after each unit.
```

### E. Status check

```
Read specs/_tracker.csv. Tell me, by owner: how many specs in
flight, how many units done / built / in_progress / blocked, and
which spec is closest to completion. Cite the row numbers.
```

### F. Mid-build spec revision

```
While building unit <unit_id> in specs/<owner>/<feature-slug>/spec.md
I discovered <issue>. Stop building. Append a numbered Revision block
to the bottom of spec.md describing the change. Re-run the Goldfish
Gate scoped to the affected units only. If this touches an
architectural invariant, revise BUILD_SPEC.md first per §9.5.
Then wait for me to re-approve.
```

### G. Just continue my work (auto-discovery)

The preferred default once a few specs are in flight. No slug needed — Claude reads the tracker and
figures it out per §9.6.

```
Read SDD_PROCESS.md. I'm <royo | surya>. Find my next pending unit
across specs/<owner>/ and specs/_tracker.csv per §9.6. Tell me in
2–3 sentences what you're about to build (Goldfish check), then
enter Build Mode for that one unit. Stop after the unit summary
block.
```

End-to-end variant:

```
... Same as above, but if I confirm the Goldfish check, build all
remaining units in that spec including wiring, without stopping
between units, until either the spec is complete or a test fails.
```

Status-only variant (no building):

```
Read specs/_tracker.csv. I'm <royo | surya>. Per §9.6, give me my
queue: per spec, units done / in_progress / not_started / blocked,
and which spec is closest to shipping. Cite row numbers.
```

---

## 16. Multi-Developer Workflow (Git)

The team operates with **two developers (royo and surya), each running their own Claude Code on
their own machine, sharing the repo via GitHub**. Claude Code reads files off disk, so the moment a
`git pull` lands new specs and tracker rows, the local Claude sees them. No special tooling beyond
git is needed; GitHub *is* the sync layer.

### 16.1 Branch model

| Lives on `main` | Lives on a feature branch |
|---|---|
| `BUILD_SPEC.md`, `SDD_PROCESS.md`, `specs/_template.md`, this whole doc tree | The code each unit produces |
| New `spec.md` **once approved** (so both devs see the contract immediately) | `BUILD_LOG.md` updates during the build session |
| Approved tracker rows in `_tracker.csv` (status `not_started` after approval) | Per-unit tracker updates (`in_progress` → `built`) — these merge back to main when the PR lands |

**Branch naming:** `<owner>/<feature-slug>`. One branch per spec. PR back to `main` when the wiring
unit is `done`.

**Why specs land on main *before* their build branch exists.** Approval is a collaborative act —
typically the owner reviews the draft and the *other* developer eyeballs it too. Putting the approved
spec on main makes it visible to everyone instantly, makes it the single canonical version, and means
the build branch starts from a known-good contract.

### 16.2 Per-session shell wrap-around

**At the start of a session:**

```bash
cd <repo>
git checkout main
git pull

# First session of this spec:
git checkout -b <owner>/<feature-slug>

# Subsequent sessions for the same spec:
git checkout <owner>/<feature-slug>
git pull --rebase origin main          # pick up any spec edits the other dev pushed
```

**Then run Claude Code** with the §15.G prompt (or §15.C if you want to be explicit about the spec).

**At the end of a session:**

```bash
git status                             # eyeball what Claude touched
git add specs/ src/ tests/             # be specific; avoid `git add .`
git commit -m "<owner>/<feature-slug>: <unit_id> — <one-line summary>"
git push -u origin <owner>/<feature-slug>
```

When the wiring unit lands `done`, open a PR. Title: the spec's `feature_title`. Body: paste the
final unit summary blocks from `BUILD_LOG.md` plus a pointer to
`specs/<owner>/<feature-slug>/spec.md`.

### 16.3 Conflict considerations

The SDD layout is intentionally low-conflict, but here's the surface area:

- **`_tracker.csv`** — the only file two developers might both edit in the same window. Mitigation:
  each owner only edits their own rows (the `owner` column makes this trivial). Different rows = git
  auto-merges. Same row = a discipline violation (one-owner-per-spec rule); resolve by talking, not
  by accepting either side blindly.
- **`spec.md`** — append-only after approval (revisions go at the bottom per §9.5). Conflicts here
  are rare and almost always indicate a process slip.
- **`BUILD_LOG.md`** — per-spec, per-owner. No shared write surface, no conflicts.
- **`BUILD_SPEC.md`** — shared, and §14 revisions are append-only for exactly this reason. Two
  concurrent architecture revisions should be a conversation, not a merge.
- **Real source code** — can conflict normally if two specs touch overlapping files. codent's
  highest-risk shared files are `src/codent/pipeline/runner.py`, `src/codent/main.py`, and
  `src/codent/models/state.py` — nearly every spec's wiring unit wants to touch at least one. Before
  approving a spec, grep other in-flight specs' `Files touched` lists for collisions on those three
  and either reorder, merge, or split the work.

### 16.4 PR review pattern

The unit summary block is what makes async review fast. The reviewer reads `BUILD_LOG.md`
top-to-bottom: every entry has the algorithm summary, files touched, and test counts. They
spot-check the actual code against the summary block — if the summary says one thing and the code
does something materially different, that's the conversation to have. **Skip line-by-line diff
reading for routine code; trust the unit blocks. Dive into the diff only when the summary smells
off.**

The **wiring test is the gate**. If `pytest tests/integration/test_<feature>_wiring.py` goes green,
the feature is mergeable. Code-style nits don't block merge — they get a follow-up issue.

### 16.5 What's *not* changed by adding a second developer

- The Goldfish Gate still runs in the authoring session, before any code (§6).
- The per-unit summary block is still mandatory and verbatim (§7.6).
- The "stop after each unit" rule still applies on each developer's machine (§7.9).
- The relationship to `BUILD_SPEC.md` is unchanged — it stays the architectural source-of-truth.

Two devs is twice the throughput, not twice the chaos — but only if both follow this doc.

---

**End of SDD_PROCESS.md.** Owners: keep this doc honest. If a part of the process stops being
useful, the doc is wrong — fix the doc before bypassing it.
