<!--
================================================================================
  codent — SDD FEATURE SPEC TEMPLATE
  Read BUILD_SPEC.md and SDD_PROCESS.md (at repo root) before filling this out.
  Replace every <placeholder> with concrete content. Delete this comment block
  before approval. Do not delete the section headings — the build-mode LLM
  depends on them.
================================================================================
-->

---
spec_id: <owner>/<feature-slug>          # e.g. royo/prefilter-gate
owner: <royo | surya>
feature_title: <Human-readable title>
status: spec_drafted                      # spec_drafted → approved → in_build → complete
created: YYYY-MM-DD
approved: ""                              # set to YYYY-MM-DD on owner approval
related_architecture_sections:            # sections of BUILD_SPEC.md this touches
  - "§<n> <section name>"
llm_calls_added: 0                        # calls this feature adds to the per-PR path (BUILD_SPEC.md §4)
---

# <Feature Title>

> **One-line elevator pitch.** What this feature does and who it's for, in a single sentence.

---

## 1. Problem

A plain-English description of the problem this feature solves. 3–5 sentences. Written for a casual
reader who has not been in the team's conversations. Avoid jargon; if jargon is unavoidable, define
it.

- *What is broken or missing today?*
- *Who is impacted?*
- *Why now?*
- *What does "good" look like after this ships?*

---

## 2. Requirements (EARS)

Each requirement gets an ID. Every unit in §6 below must cite the requirement IDs it satisfies. No
orphan requirements; no orphan units.

| ID | Form | Requirement |
|----|------|-------------|
| R1 | Ubiquitous | The `<system>` shall `<response>`. |
| R2 | Event-driven | When `<trigger>`, the `<system>` shall `<response>`. |
| R3 | State-driven | While `<state>`, the `<system>` shall `<response>`. |
| R4 | Optional | Where `<feature>` is included, the `<system>` shall `<response>`. |
| R5 | Unwanted | If `<trigger>`, then the `<system>` shall `<response>`. |

*(Drop the rows you don't need. Add more as needed. Keep the table format.)*

---

## 3. Technical Plan

A jargon-light prose description of how the feature is built. Include a block diagram (ASCII is
fine) of the major moving parts and the data/control flow between them. Where the feature interacts
with the existing architecture — a pipeline stage, the `PipelineState` contract, an interface
(`LLMProvider` / `SandboxRunner` / `Embedder` / `Linter`), the config layers, the GitHub client —
say so explicitly and reference the relevant section of `BUILD_SPEC.md`.

```
[ascii block diagram here]
```

**Key design decisions:**

- *Decision 1:* `<what was chosen and why>`. Alternatives considered are in §4.
- *Decision 2:* `<...>`

**External dependencies introduced (if any):** list new pip packages with versions and a one-line
justification per package. If none, write "None — uses existing project dependencies only."

**Data contract changes (if any):** list new or changed Pydantic models. If this feature adds a
field to `PipelineState`, name it and say which stage owns it — no two stages may write the same
field (`BUILD_SPEC.md §7`).

---

## 3a. Architectural Invariant Check

Answer every row. This is the cheapest place to catch a spec that fights the architecture.

| Invariant | Ref | This spec's answer |
|---|---|---|
| **LLM call budget.** How many calls does this add to the per-PR path? If >0, is it conditional, and is it batched (never per-file/per-finding)? | `BUILD_SPEC.md §4` | `<answer>` |
| **Interface rule.** Does anything under `pipeline/` import a concrete implementation instead of an interface? | `§6` | `<answer>` |
| **State ownership.** Does any stage write a `PipelineState` field another stage owns? | `§7` | `<answer>` |
| **Line-number rule.** If this produces findings, are out-of-diff lines dropped rather than repaired? | `§7` | `<answer>` |
| **Risk formula.** Does this change any weight or band? (If yes, `BUILD_SPEC.md §14` revision required first.) | `§9` | `<answer>` |
| **Sandbox posture.** Does this widen what sandboxed code can reach? | `§10` | `<answer>` |
| **Config restriction.** Does this add a `.codent.yml` key that is anything other than a number or a path glob? | `§11` | `<answer>` |

If any answer is a violation, **stop** — the architecture is revised first (`SDD_PROCESS.md §9.5`),
not worked around.

---

## 4. Alternatives Considered (and Rejected)

For each major decision in §3, list the options that were considered but not chosen, and why. Even a
brief "we considered X, rejected because Y" prevents the AI from drifting back to those rejected
choices mid-build.

- **Alternative A:** `<description>` — *Rejected because:* `<reason>`.
- **Alternative B:** `<description>` — *Rejected because:* `<reason>`.

---

## 5. Scope

**In scope (this spec):**
- `<bullet>`

**Out of scope (explicitly):**
- `<bullet>` — *deferred to <future-spec | post-MVP | won't do>*

---

## 6. Unit Decomposition

This is the heart of the spec. Each unit is something the LLM can build and test atomically in one
short build cycle, *without depending on a not-yet-built unit being present*.

Order units so that dependencies flow top-to-bottom (a unit may depend on prior units; never on
later ones).

### Unit `unit-01` — `<Short imperative title>`

- **Purpose.** *What this unit accomplishes, in 1–2 sentences. The "why" not just the "what".*
- **Satisfies requirements:** R`<n>`, R`<m>`
- **Inputs.**
  - `<param_name>: <type>` — `<meaning>`
- **Outputs.**
  - `<return_or_side_effect>: <type>` — `<meaning>`
- **Algorithm / Logic.** *Plain-English prose describing how the unit computes its output. Include
  any non-obvious choices (data structures, edge-case handling, ordering, idempotency). This is the
  field that, when copied to the tracker's `algorithm_summary`, prevents the feature from being a
  black box.*
  ```
  pseudocode allowed, but keep it short — prose is the contract,
  pseudocode is the illustration.
  ```
- **Files touched.**
  - `<path/to/file>` — *new | modified* — `<one-line role>`
  - `<...>`
- **Test cases.** *Each test case = an `input → expected output` statement. The build-mode LLM
  converts each into a real test. Remember `SDD_PROCESS.md §12`: no real LLM, GitHub, or sandbox
  calls in unit tests — inject fakes through `Deps`.*
  - `<test_name>`: given `<input>`, the unit returns/raises/produces `<expected>`. *Test file:* `tests/unit/test_<unit>.py`
  - `<test_name>`: edge case — `<...>`. *Test file:* `<same path>`
  - `<test_name>`: invalid input — `<...>`. *Test file:* `<same path>`
- **Acceptance criteria.**
  - [ ] All declared test cases pass.
  - [ ] No file outside the `Files touched` list was modified.
  - [ ] No new dependency beyond those declared in §3.
  - [ ] `<any unit-specific criterion — e.g. no LLM call, deterministic output, latency under X ms>`

### Unit `unit-02` — `<...>`

*(same structure)*

### Unit `unit-NN` — `<...>`

*(repeat as needed; keep unit ids zero-padded for sortability — `unit-01`, `unit-02`, …)*

---

## 7. Wiring Plan

How the units above compose into the assembled feature. Include:

- **Composition order.** Which unit is invoked / registered / called by which.
- **Integration files.** The files the wiring unit will touch to make the assembled feature visible
  to its consumer — typically a stage registration in `src/codent/pipeline/runner.py`, a route in
  `src/codent/main.py`, a field on `src/codent/models/state.py`, or a dependency added to `Deps`.
  *(These three files are the repo's highest-conflict paths — `SDD_PROCESS.md §16.3`.)*
- **Data flow at the seam.** What crosses the boundary between this feature and the rest of the
  system — which `PipelineState` fields are read, which are written, what HTTP shape, what event.
- **Failure modes at the seam.** What happens if an upstream stage failed or produced an empty
  result? What does this feature do on partial failure? Mirror the spec's R5 (unwanted-behaviour)
  rows. codent's default posture: a stage failure degrades the review, it does not crash the job —
  record a `StageError` and continue.

```
[ascii diagram of the wired-up feature, showing each unit and the seams]
```

---

## 8. Wiring Test

The end-to-end test for the assembled feature. Executes against the real, wired-up code — *no mocks
of the units inside the feature*. External services (Anthropic, GitHub, the sandbox) are faked at
the system boundary through `Deps`, per `SDD_PROCESS.md §12`.

- **Test file:** `tests/integration/test_<feature>_wiring.py`
- **What it does:**
  1. `<step>` — e.g. *builds a `PipelineState` from `tests/fixtures/<sample>.diff`*
  2. `<step>` — e.g. *runs the pipeline with a fake `LLMProvider` scripted to return `<...>`*
  3. `<step>` — e.g. *asserts the resulting `ScoredFinding` list matches `<...>`*
- **Pass criteria:** `<the exact assertion(s) that must hold>`
- **LLM budget assertion:** `assert state.llm_calls_made == <n>` — required if this test runs the
  pipeline end-to-end (`SDD_PROCESS.md §12`, rule 4). Otherwise write "N/A — does not run the
  pipeline."
- **Run command:** `pytest tests/integration/test_<feature>_wiring.py -v`

---

## 9. Risks & Open Questions

A short list of risks specific to this feature and any questions the developer wants flagged for
owner-only review (do not let the LLM silently invent answers). Each open question should have a
`resolved_by` placeholder for when it's answered.

- **Risk:** `<...>` — *Mitigation:* `<...>`
- **Open question:** `<...>` — *Resolved by:* `_<owner | YYYY-MM-DD>_`

---

## 10. Goldfish Checklist

The authoring LLM fills this in *after* its cold-read self-check (see `SDD_PROCESS.md §6`) and before
handing the spec back for approval. Each item must be answered with a concrete reference into the
spec (section/unit ID), not "yes".

- [ ] **Comprehension.** A fresh LLM reading this spec from cold can state, in their own words, what
      the feature does and how it works. — *Demonstrated in:* `<section refs>`
- [ ] **Critic.** All "first-pass" critiques an expert reviewer would raise have been addressed.
      Remaining nits, if any, are listed under §9. — *Demonstrated in:* `<section refs>`
- [ ] **Implementation-ready.** No build-time decision is left to invention. Every unit has its
      algorithm, files, and tests declared. — *Demonstrated in:* §6 (all units)
- [ ] **Unit atomicity.** No unit depends on a later unit. Each unit is testable in isolation as
      written. — *Demonstrated in:* §6 ordering
- [ ] **Requirement coverage.** Every requirement ID in §2 is cited by at least one unit in §6. —
      *Demonstrated in:* unit "Satisfies requirements" lines
- [ ] **No-orphan files.** Every file in `Files touched` across §6 + §7 is reachable from a use case
      in §1 or a requirement in §2. — *Demonstrated in:* `<your audit summary>`
- [ ] **Reaches a real PR.** If this feature produces output a human is supposed to see, some unit's
      `Files touched` names the path from the pipeline to the posted GitHub comment — and if it adds
      a pipeline stage, `pipeline/runner.py` appears in §7's integration files. A stage nothing calls
      is a stage nobody gets. — *Demonstrated in:* `<the unit and file that wire it, or the decision
      to defer>`
- [ ] **Invariant check complete.** Every row of §3a is answered, and no answer is a violation. —
      *Demonstrated in:* §3a
- [ ] **Anti-pattern check.** None of the items in `SDD_PROCESS.md §14` are present in this spec (no
      skipped tests, no missing wiring test, no opportunistic refactors, no unbudgeted LLM call, no
      concrete implementation imported inside `pipeline/`).

---

## 11. Revisions

*Append-only. Do not edit the body of an approved spec; add a revision block here instead.*

<!--
### Revision 1 — YYYY-MM-DD by <owner>
**Reason:** <what we learned>
**Change:** <what we changed about which units>
**Affected units:** <list>
**BUILD_SPEC.md revision required:** <no | yes — §<n>, landed YYYY-MM-DD>
**Re-approved on:** YYYY-MM-DD
-->

---

**End of spec.** When approved, run the prompt in `SDD_PROCESS.md §15.B` to start the build.
