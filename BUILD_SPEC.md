# 🧭 codent — BUILD SPEC

**Version:** 1.0
**Status:** Active. This is the **architectural source-of-truth** for `codent`.
**Owners:** royo, surya
**Supersedes:** `ARCHITECTURE_PROPOSAL.md` (kept in the repo as the decision history that produced this doc).

> **One-line summary.** `codent` is an AI pull-request reviewer: a GitHub App receives a PR
> webhook, a deterministic pipeline gathers evidence (linters, an AST index, a RAG index, and
> generated tests executed in a sandbox), an LLM reasons over that evidence, and a deterministic
> risk score decides which findings are confident enough to post as inline GitHub comments.
> **The LLM is called at most four times per PR, by design.**

This document holds the vision, the stack, the pipeline, the data contracts, and the design
decisions. It does **not** hold feature specs — those live under `specs/<owner>/` and are governed
by `SDD_PROCESS.md`. Read this file once per session to ground yourself; read `SDD_PROCESS.md`
every time you do feature work.

---

## 📌 TABLE OF CONTENTS

1. [Specs Index](#1-specs-index)
2. [Vision & Product Boundary](#2-vision--product-boundary)
3. [The Pipeline](#3-the-pipeline)
4. [LLM Call Budget](#4-llm-call-budget)
5. [Stack](#5-stack)
6. [Repository Layout](#6-repository-layout)
7. [Core Data Contracts](#7-core-data-contracts)
8. [Stage Reference](#8-stage-reference)
9. [Risk Scoring & the Confidence Gate](#9-risk-scoring--the-confidence-gate)
10. [Sandbox & Threat Model](#10-sandbox--threat-model)
11. [Configuration](#11-configuration)
12. [Design Decisions & Tradeoffs](#12-design-decisions--tradeoffs)
13. [Deferred / Out of Scope](#13-deferred--out-of-scope)
14. [Revisions](#14-revisions)

---

## 1. Specs Index

The live, unit-level status of all work is in **`specs/_tracker.csv`**. That CSV is the index —
do not build a second one. This section carries one line per spec for human orientation.

| Spec ID | Owner | Feature | Status |
|---|---|---|---|
| _(none yet)_ | — | — | — |

*Update this table when a spec is approved and when it completes.*

---

## 2. Vision & Product Boundary

### What codent is

A reviewer that earns trust by **only speaking when it has evidence**. Most AI PR reviewers fail
in one of two directions: they comment on everything (noise, reviewers mute the bot) or they run
one big LLM call over the whole diff (expensive, hallucinated line numbers, no way to tell a real
bug from a plausible sentence). codent's answer to both is the same: gather deterministic evidence
first, spend LLM calls only where the evidence says something interesting might be happening, and
gate every posted comment behind a numeric confidence score that a human can tune.

### What "complete product" means here

- It installs as a **GitHub App**, not a script someone runs by hand.
- It **generates and runs tests** against the change, so at least some findings carry executable
  proof rather than an opinion.
- It **reconstructs what the PR was supposed to do** and checks the diff against that, instead of
  only asking "is this code good".
- Its output is **scored**, not labelled — a 0–100 risk number with a posting threshold, so the
  signal-to-noise ratio is a config knob rather than a rewrite.

### Explicit non-goals

- Not a linter. codent runs linters as an *input*; it does not replace them or restate their output.
- Not a style enforcer. Formatting and naming opinions are out of scope; that's what formatters and
  CI are for.
- Not an auto-fixer. codent posts findings, it does not push commits. (See §13.)

### Language scope for v1

**Python repositories only.** The AST layer (`ast_parser`) and the linter layer are Python-specific
in v1. Multi-language support is a deliberate later expansion (see §13) — every stage that touches
language specifics must be written behind an interface so that expansion is additive, not a rewrite.

---

## 3. The Pipeline

```
 GitHub App (webhook: pull_request opened/synchronize)
        │
        ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ QUEUE — per-repo, async, at-most-one-in-flight per PR        │
 │ (a new push to the same PR cancels the older job)            │
 └──────────────────────────┬───────────────────────────────────┘
                            ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ STAGE 1 — PR INTAKE                                          │
 │  • fetch diff, changed-file list, PR body, linked issue,     │
 │    commit messages                                           │
 │  • Intent Reconstruction  ⟵ LLM CALL #1 (every PR)           │
 │    → IntentSpec: a structured list of what this PR claims    │
 │      it does (requirements the diff should satisfy)          │
 └──────────────────────────┬───────────────────────────────────┘
                            ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ STAGE 2 — REPO ANALYSIS  (no LLM calls, cached per commit)   │
 │  • AST index: symbol table, call graph, import graph         │
 │  • RAG index: per-repo vector collection of code chunks      │
 │  Both built once per repo, invalidated by base-branch SHA    │
 └──────────────────────────┬───────────────────────────────────┘
                            ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ STAGE 3 — PRE-FILTER GATE  (no LLM calls)  ⟵ THE COST LEVER  │
 │  Per changed file, deterministically:                        │
 │    • run linters                                             │
 │    • run AST existence-checks (undefined symbols, arity      │
 │      mismatches, dead imports, signature drift at call sites)│
 │  If a file has ZERO findings AND its diff is below the        │
 │  triviality threshold → the file is DROPPED. It never        │
 │  reaches an LLM.                                             │
 │  If NO file survives → pipeline ends. Post nothing (or a      │
 │  one-line "no findings" summary if configured).              │
 └──────────────────────────┬───────────────────────────────────┘
                            ▼  (surviving files only)
 ┌──────────────────────────────────────────────────────────────┐
 │ STAGE 4 — TARGETED TEST GEN + EXECUTION  ⟵ LLM CALL #2       │
 │  One batched call generates tests for ALL surviving files,   │
 │  aimed at the IntentSpec's stated requirements.              │
 │  Tests run in a sandbox (§10). Capture pass/fail + stderr    │
 │  as EVIDENCE, not as a verdict.                              │
 └──────────────────────────┬───────────────────────────────────┘
                            ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ STAGE 5 — REASONING PASS A  ⟵ LLM CALL #3                    │
 │  Input: diff hunks + AST FACTS BLOCK (pre-computed, stated   │
 │  as fact — never asked to be derived) + RAG neighbours +     │
 │  IntentSpec + test results.                                  │
 │  Output: candidate Findings, each with a self-reported       │
 │  certainty and a cited evidence list.                        │
 └──────────────────────────┬───────────────────────────────────┘
                            ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ STAGE 6 — ADAPTIVE REASONING PASS B  ⟵ LLM CALL #4           │
 │  CONDITIONAL. Runs only for findings in the borderline band  │
 │  (§9). A finding backed by a failing test + a lint error +   │
 │  a hard AST fact already agrees with itself — asking twice   │
 │  buys nothing. One batched call covers all borderline        │
 │  findings; if there are none, the call never happens.        │
 │  Prompted to REFUTE, not to re-review.                       │
 └──────────────────────────┬───────────────────────────────────┘
                            ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ STAGE 7 — RISK SCORING  (deterministic, no LLM)              │
 │  Combine test signal / A-B agreement / lint severity /       │
 │  AST signal into a 0–100 score per finding. §9.              │
 └──────────────────────────┬───────────────────────────────────┘
                            ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ STAGE 8 — CONFIDENCE GATE + POST  (deterministic)            │
 │  Findings with score ≥ threshold become inline comments.     │
 │  Below-threshold findings are logged, never posted.          │
 │  Plus one summary comment with the PR-level verdict.         │
 └──────────────────────────────────────────────────────────────┘
```

### Why this order

Every stage before Stage 4 is free. The pipeline spends the cheap deterministic signals first
precisely so that the expensive stages see a smaller, pre-qualified input. The pre-filter gate is
not an optimization bolted onto a working pipeline — it is a **first-class stage that the rest of
the design assumes exists**, which is why Stages 4–6 are written to operate on "surviving files"
rather than "the diff".

---

## 4. LLM Call Budget

This is a hard architectural constraint, not an aspiration. **Any spec that adds a fifth
unconditional LLM call to the per-PR path must amend this section first.**

| # | Stage | When it fires | Batched? |
|---|---|---|---|
| 1 | Intent reconstruction | Every PR | n/a — one call |
| 2 | Test generation | Only if ≥1 file survives the pre-filter gate | Yes — one call for all surviving files |
| 3 | Reasoning Pass A | Only if ≥1 file survives the pre-filter gate | Yes — one call for all surviving files |
| 4 | Reasoning Pass B | Only if ≥1 finding lands in the borderline band | Yes — one call for all borderline findings |

**Per-PR totals:**

- Trivial/clean PR (nothing survives the gate): **1 call**
- Normal PR with clear-cut findings: **3 calls**
- Worst case: **4 calls**

**The batching rule.** Stages 4, 5, and 6 each make *one* call covering all their inputs, never one
call per file or per finding. Per-file fan-out is the single most likely way this budget silently
degrades — a spec that introduces it is violating this section even if each individual call looks
cheap.

**The counting rule.** "LLM call" means one request to the provider that consumes input tokens for
reasoning. Retries after a transport error are not new calls. A call that is skipped because its
gate did not open is not a deferred call — it simply does not exist.

---

## 5. Stack

| Layer | Choice | Notes |
|---|---|---|
| Language | **Python 3.11** | |
| Web framework | **FastAPI** + uvicorn | Webhook receiver, health endpoints, admin routes |
| Orchestration | **Plain async stage functions** over a Pydantic `PipelineState` | **No LangGraph.** See §12.1 |
| Vector store | **ChromaDB** (persistent, local dir) | One collection per repo |
| Embeddings | **CodeBERT** via `sentence-transformers` | Behind an `Embedder` interface (§12.3) |
| AST | Python stdlib **`ast`** | Behind an `ASTIndexer` interface |
| Linters | **ruff** (+ `mypy` where the repo already configures it) | Behind a `Linter` interface |
| LLM | **Anthropic Claude**, model `claude-opus-5` | Behind an `LLMProvider` interface (§12.2) |
| Sandbox | **Local subprocess runner** for now | Behind a `SandboxRunner` protocol; Docker/Railway later (§10) |
| GitHub | **PyGithub** / `githubkit` for the App + REST calls | Webhook signature verification is mandatory |
| Tests | **pytest** + `pytest-asyncio` | See `SDD_PROCESS.md §12` |
| Config | **pydantic-settings** (env) + per-repo `.codent.yml` | §11 |

**Anthropic call conventions** (apply to every LLM call in the pipeline):

- Model: `claude-opus-5` (configurable via `CODENT_LLM_MODEL`).
- Use adaptive thinking: `thinking={"type": "adaptive"}`. Do **not** pass `budget_tokens` — it is
  rejected with a 400 on this model.
- Use `output_config={"effort": ...}` to tune depth per stage rather than switching models:
  `low` for intent reconstruction, `high` for Pass A, `high` for Pass B.
- Use **structured outputs** (`output_config.format`) for every stage — each stage's output is a
  typed Pydantic model (§7), never free text the code has to parse.
- Stream (`client.messages.stream(...)`) for the reasoning passes, which can produce long output.
- Always check `response.stop_reason` before reading content.

---

## 6. Repository Layout

```
codent/
├── BUILD_SPEC.md               # this file — architectural source-of-truth
├── SDD_PROCESS.md              # the methodology every feature follows
├── ARCHITECTURE_PROPOSAL.md    # historical: the draft this doc resolved
├── pyproject.toml
├── .env.example
├── specs/
│   ├── _template.md            # per-feature spec template
│   ├── _tracker.csv            # unit-level progress across all specs — THE index
│   ├── royo/README.md
│   └── surya/README.md
├── src/codent/
│   ├── main.py                 # FastAPI app + route registration
│   ├── config.py               # Settings (env) + per-repo .codent.yml merge
│   ├── models/                 # Pydantic data contracts (§7)
│   │   ├── state.py            #   PipelineState
│   │   ├── intent.py           #   IntentSpec
│   │   ├── evidence.py         #   LintFinding, ASTFact, RAGNeighbour, TestResult
│   │   └── finding.py          #   Finding, ScoredFinding
│   ├── webhook/
│   │   ├── router.py           # POST /webhook — signature verify, enqueue, 202
│   │   └── signature.py        # HMAC verification
│   ├── queue/
│   │   └── runner.py           # per-repo async queue, per-PR cancellation
│   ├── pipeline/
│   │   ├── runner.py           # the stage chain + error containment
│   │   ├── stage_01_intake.py
│   │   ├── stage_02_index.py
│   │   ├── stage_03_prefilter.py
│   │   ├── stage_04_testgen.py
│   │   ├── stage_05_reason_a.py
│   │   ├── stage_06_reason_b.py
│   │   ├── stage_07_risk.py
│   │   └── stage_08_post.py
│   ├── analysis/
│   │   ├── ast_index.py        # symbol table, call graph, import graph
│   │   ├── ast_facts.py        # the pre-computed FACTS BLOCK builder
│   │   ├── rag.py              # ChromaDB collection lifecycle + retrieval
│   │   ├── embedder.py         # Embedder interface + CodeBERT impl
│   │   └── linters.py          # Linter interface + ruff/mypy impls
│   ├── sandbox/
│   │   ├── base.py             # SandboxRunner protocol + SandboxResult
│   │   └── local.py            # subprocess + rlimit implementation
│   ├── llm/
│   │   ├── provider.py         # LLMProvider interface
│   │   ├── anthropic_provider.py
│   │   └── prompts/            # one file per stage prompt
│   ├── scoring/
│   │   ├── risk.py             # the 0–100 formula (§9)
│   │   └── gate.py             # threshold resolution + filtering
│   └── github/
│       ├── client.py           # App auth, installation tokens
│       └── comments.py         # inline comment + summary posting
└── tests/
    ├── unit/                   # tests/unit/test_<unit>.py
    ├── integration/            # tests/integration/test_<feature>_wiring.py
    └── fixtures/               # sample diffs, sample repos, recorded payloads
```

**The interface rule.** Every box in §5 marked "behind an interface" exists so that a later spec can
swap the implementation without touching the pipeline. A spec that imports a concrete implementation
(`CodeBERTEmbedder`, `LocalSandboxRunner`, `AnthropicProvider`) anywhere inside `pipeline/` is
violating this rule — the pipeline depends on the interface, and the concrete choice is wired in
`main.py`.

---

## 7. Core Data Contracts

These are the shapes every stage reads and writes. A stage's job is: take `PipelineState`, add its
own field, return it. **Stages never mutate a field another stage owns.**

```python
# models/state.py
class PipelineState(BaseModel):
    # --- set at intake ---
    repo_full_name: str            # "owner/repo"
    pr_number: int
    head_sha: str
    base_sha: str
    diff: Diff                     # parsed, per-file, per-hunk with line numbers
    pr_body: str
    linked_issue_body: str | None
    commit_messages: list[str]

    # --- stage 1 ---
    intent: IntentSpec | None = None

    # --- stage 2 ---
    ast_index_id: str | None = None    # handle into the cached index
    rag_collection: str | None = None

    # --- stage 3 ---
    surviving_files: list[str] = []
    lint_findings: list[LintFinding] = []
    ast_facts: list[ASTFact] = []

    # --- stage 4 ---
    test_results: list[TestResult] = []

    # --- stage 5 / 6 ---
    findings: list[Finding] = []

    # --- stage 7 / 8 ---
    scored: list[ScoredFinding] = []
    posted: list[int] = []             # GitHub comment IDs

    # --- cross-cutting ---
    llm_calls_made: int = 0            # asserted against §4 in the wiring test
    stage_errors: list[StageError] = []
```

```python
# models/intent.py
class IntentRequirement(BaseModel):
    id: str                  # "I1", "I2", ...
    statement: str           # "the endpoint shall reject requests without a token"
    source: Literal["pr_body", "linked_issue", "commit_message", "inferred"]

class IntentSpec(BaseModel):
    summary: str
    requirements: list[IntentRequirement]
    confidence: float        # 0–1, the model's own read on how clear the PR's stated intent was
```

```python
# models/evidence.py
class LintFinding(BaseModel):
    file: str; line: int; rule: str
    severity: Literal["error", "warning", "info"]
    message: str

class ASTFact(BaseModel):
    file: str; line: int
    kind: Literal["undefined_symbol", "arity_mismatch", "signature_drift",
                  "unused_import", "no_callers", "symbol_exists"]
    hardness: Literal["hard", "soft"]     # drives the AST term in §9
    statement: str                        # stated to the LLM as fact, not asked

class RAGNeighbour(BaseModel):
    file: str; start_line: int; end_line: int
    snippet: str; similarity: float

class TestResult(BaseModel):
    target_file: str
    satisfies_intent: list[str]           # IntentRequirement ids
    test_source: str
    outcome: Literal["passed", "failed", "errored", "timed_out", "not_run"]
    stderr: str
    deterministic: bool                   # True only if a re-run reproduced the same outcome
```

```python
# models/finding.py
class Finding(BaseModel):
    id: str
    file: str; line: int                  # must map to a line present in the diff
    title: str
    body: str
    category: Literal["correctness", "security", "intent_mismatch",
                      "api_contract", "test_gap"]
    pass_a_certainty: float               # 0–1, self-reported by the model
    evidence: list[str]                   # ids of the lint/ast/rag/test items cited
    pass_b_verdict: Literal["corroborated", "refuted", "not_run"] = "not_run"

class ScoredFinding(Finding):
    risk_score: int                       # 0–100, from §9
    posted: bool
```

**Line-number rule.** A `Finding` whose `line` does not fall inside a hunk of the current diff is
**dropped before scoring**, not repaired. Comments on lines the PR did not touch are the fastest way
to lose a reviewer's trust, and a model that cites an out-of-diff line has usually hallucinated the
context too.

---

## 8. Stage Reference

Each stage is an `async def stage_xx(state: PipelineState, deps: Deps) -> PipelineState`. `Deps`
carries the interface instances (`LLMProvider`, `SandboxRunner`, `Embedder`, GitHub client), which
is what makes every stage unit-testable with fakes and no network.

### Stage 1 — PR Intake
Fetches the PR, parses the diff into per-file/per-hunk structures with real line numbers, pulls the
linked issue body if the PR references one, then makes **LLM call #1** to produce an `IntentSpec`.
Runs for every PR regardless of size. If the PR body is empty and no issue is linked, the call still
runs but is expected to return a low-confidence `IntentSpec` derived from commit messages and the
diff shape — downstream stages must handle `intent.confidence` being low rather than assuming a
useful spec.

### Stage 2 — Repo Analysis
Builds (or reuses) the AST index and the RAG collection for the repo at `base_sha`. Both are cached
and keyed by `(repo_full_name, base_sha)`. A cache hit is the normal case for the second and later
PRs against the same base. **No LLM calls.** This stage is idempotent and safe to re-run.

### Stage 3 — Pre-filter Gate
For each changed file: run the linters, run the AST existence checks, and compute the diff's
size. A file survives if it has at least one lint finding, at least one AST fact of any hardness, or
a changed-line count at or above `triviality_threshold` (default 25 — §11). Everything else is
dropped with a logged reason. If nothing survives, the pipeline short-circuits to Stage 8 with an
empty finding list. **No LLM calls.**

### Stage 4 — Targeted Test Generation + Execution
**LLM call #2**, one batched request covering every surviving file. The prompt carries the
`IntentSpec` requirements and asks for tests that would fail if a stated requirement is not met.
Generated tests run through the `SandboxRunner` (§10). Each test is run **twice**; `deterministic`
is set only if both runs agree, and a non-deterministic test's result is downgraded to `errored`
so it cannot drive a high risk score. Test output is evidence for Stage 5, never a verdict on its
own.

### Stage 5 — Reasoning Pass A
**LLM call #3**, one batched request. Assembles the prompt from: the diff hunks, the **AST facts
block** (stated declaratively — "`process_batch` takes 2 positional args; the call at line 88 passes
3" — never "check whether the arity matches"), the top-k RAG neighbours per hunk, the `IntentSpec`,
and the test results. Returns candidate `Finding`s with structured output. Findings citing no
evidence at all are dropped here.

### Stage 6 — Adaptive Reasoning Pass B
**LLM call #4**, conditional and batched. Only findings whose Stage-7 *provisional* score (computed
with `agreement` held at its neutral 0.5) lands in the borderline band `[borderline_low,
borderline_high)` are sent. The prompt's job is **refutation**: given the same evidence, argue the
finding is wrong; default to `refuted` when uncertain. This asymmetry is deliberate — Pass B exists
to kill plausible-but-wrong findings, not to add new ones. **Pass B never introduces findings.**

### Stage 7 — Risk Scoring
Pure function, no I/O, no LLM. See §9. This is the most heavily unit-tested module in the codebase
because it is the only place where all four evidence channels meet.

### Stage 8 — Confidence Gate + Post
Resolves the threshold (§11), filters, and posts. Inline comments carry the finding body plus a
one-line evidence trail ("generated test `test_rejects_missing_token` failed; ruff E501 at this
line; `validate()` has no callers"). One summary comment carries the PR-level verdict, the counts,
and — importantly — **how many findings were suppressed below threshold**, so a reviewer can tell
the difference between "found nothing" and "found nothing confident".

---

## 9. Risk Scoring & the Confidence Gate

### The formula

```
risk_score = round(100 * (
      0.40 * test_signal
    + 0.25 * agreement
    + 0.20 * lint_severity
    + 0.15 * ast_signal
))
```

Each term is normalized to `[0.0, 1.0]`:

| Term | Weight | 1.0 | 0.5 | 0.0 |
|---|---|---|---|---|
| `test_signal` | **0.40** | A generated test targeting this finding failed **deterministically** | Test errored, timed out, or was non-deterministic | Test passed, or no test targeted this finding |
| `agreement` | **0.25** | Pass B corroborated, **or** Pass A certainty ≥ `pass_a_high` and Pass B was correctly skipped | Pass B was not run and Pass A was borderline | Pass B refuted |
| `lint_severity` | **0.20** | A lint finding of severity `error` on the same file within ±2 lines | severity `warning` on the same span | `info`, or no lint finding on the span |
| `ast_signal` | **0.15** | A `hard` ASTFact on the same span (undefined symbol, arity mismatch, signature drift) | a `soft` ASTFact (no_callers, unused_import) | no ASTFact on the span |

**Why these weights.** A failing test is the only channel that constitutes *executable proof*, so it
carries the largest single share — but deliberately less than half, so a flaky or badly-generated
test can never post a comment on its own. Agreement is second because it is the only channel that
can actively *veto*: `agreement = 0` (Pass B refuted) subtracts 25 points, which drops a finding
from "would post" to "would not post" across the default threshold from almost any starting point.
Lint and AST are weighted below the LLM-derived channels because they are already visible to the
developer through their own tooling — their value here is corroboration, not novelty.

**Rounding is to the nearest integer.** Scores are integers 0–100 in every log, comment, and test
assertion, so that test expectations are exact rather than float-comparison-fuzzy.

### The bands

| Constant | Default | Meaning |
|---|---|---|
| `confidence_threshold` | **55** | Findings scoring `>= 55` are posted |
| `borderline_low` | **40** | Below this, Pass B is not worth a call — the finding will not clear the gate anyway |
| `borderline_high` | **70** | At or above this, the evidence already agrees strongly — Pass B buys nothing |
| `pass_a_high` | **0.8** | Pass A self-certainty at or above this counts as "confident" for the `agreement` term |

**Worked examples** (these belong in `tests/unit/test_risk.py` as-is):

- *Failing deterministic test + lint error + hard AST fact, Pass B skipped as too-strong, Pass A
  certainty 0.9:* `100*(0.40*1.0 + 0.25*1.0 + 0.20*1.0 + 0.15*1.0)` = **100** → posted.
- *Failing deterministic test only, nothing else, Pass A borderline, Pass B not run:*
  `100*(0.40*1.0 + 0.25*0.5 + 0 + 0)` = **53** → **not** posted (just below 55). This is intentional:
  one channel alone is not enough.
- *Same as above but Pass B corroborates:* `100*(0.40 + 0.25*1.0)` = **65** → posted.
- *Same but Pass B refutes:* `100*(0.40 + 0)` = **40** → not posted.
- *Lint warning + soft AST fact, no test, Pass B refuted:* `100*(0 + 0 + 0.20*0.5 + 0.15*0.5)` =
  **18** → not posted.

### Tuning discipline

The weights above are the **spec'd default**, not a permanent truth — but they are a *contract*.
Changing a weight requires a revision block on this file (§14), because every risk-scoring test
asserts against these numbers. Changing the *threshold* requires nothing but config (§11), which is
the intended tuning surface for day-to-day noise control.

---

## 10. Sandbox & Threat Model

### What we are defending against

Stage 4 executes **code derived from a pull request**, in a process we control, against a checkout
of the repo. The PR author is untrusted — an open-source repo accepts PRs from anyone, and a PR is
an ideal delivery vehicle for code that wants to read our environment. Concretely, the threats are:

1. **Credential theft** — generated or imported code reads `ANTHROPIC_API_KEY`, the GitHub App
   private key, or the installation token out of the environment or the filesystem.
2. **Exfiltration** — anything the process learns leaves over the network.
3. **Host damage / persistence** — writes outside the checkout, or leaves something running.
4. **Resource exhaustion** — an infinite loop or a fork bomb starves the service.
5. **Pivot** — the test process reaches other services on the local network.

### v1 posture: local subprocess runner

Per the owners' decision, v1 runs tests **locally, in a constrained subprocess**, because the
project is in local development and there is no deployment target yet. This is an accepted,
explicitly time-boxed risk, and the mitigations below are **mandatory in v1**, not deferred:

- Run as a **separate, unprivileged OS user** — never the user running the app.
- **Scrubbed environment.** The child process gets an explicit allowlist of env vars. No secret in
  the parent's environment is inherited, ever. This is the single most important control.
- **`rlimit` caps**: CPU time, address space, file size, process count (`RLIMIT_NPROC` to stop fork
  bombs), and open file descriptors.
- **Wall-clock timeout** with a hard kill of the whole process group.
- **Working directory is a fresh temp checkout**, deleted afterwards, and the only writable path.
- **No network** to the extent the platform allows it (best-effort in v1 — see the honest limit
  below).

**The honest limit.** A subprocess with rlimits is a *resource* boundary, not a *security* boundary.
It does not reliably prevent a determined attacker from making an outbound network connection on a
developer machine. v1 is safe enough for local development against repos the owners control; it is
**not** safe to point at arbitrary public PRs. That transition is gated on the container backend
below.

### The interface

```python
# sandbox/base.py
class SandboxResult(BaseModel):
    outcome: Literal["passed", "failed", "errored", "timed_out"]
    stdout: str; stderr: str; exit_code: int; duration_s: float

class SandboxRunner(Protocol):
    async def run(self, *, checkout_dir: Path, test_files: dict[str, str],
                  command: list[str], timeout_s: int) -> SandboxResult: ...
```

Stage 4 depends only on this protocol. `LocalSandboxRunner` is v1; `DockerSandboxRunner` (ephemeral
container, `--network=none`, read-only mount, non-root, dropped capabilities) and a Railway-hosted
runner are later specs that implement the same protocol and change **no pipeline code**.

**Promotion criterion.** Before codent is pointed at any repo whose PRs are not authored by the
owners, the Docker backend must be the default. That is a hard gate, recorded here so it does not
get lost.

---

## 11. Configuration

Two layers, merged with **per-repo winning over env**:

### Layer 1 — environment (`pydantic-settings`, prefix `CODENT_`)

| Var | Default | Meaning |
|---|---|---|
| `CODENT_LLM_MODEL` | `claude-opus-5` | Reasoning model |
| `CODENT_CONFIDENCE_THRESHOLD` | `55` | Global posting threshold |
| `CODENT_BORDERLINE_LOW` | `40` | Pass B band floor |
| `CODENT_BORDERLINE_HIGH` | `70` | Pass B band ceiling |
| `CODENT_TRIVIALITY_THRESHOLD` | `25` | Changed lines below which a clean file is dropped |
| `CODENT_SANDBOX_BACKEND` | `local` | `local` \| `docker` |
| `CODENT_SANDBOX_TIMEOUT_S` | `60` | Per-test-run wall clock |
| `CODENT_RAG_TOP_K` | `5` | Neighbours retrieved per hunk |
| `CODENT_POST_SUMMARY_WHEN_EMPTY` | `false` | Post a "no findings" summary on clean PRs |
| `ANTHROPIC_API_KEY` | — | |
| `CODENT_GITHUB_APP_ID` / `_PRIVATE_KEY` / `_WEBHOOK_SECRET` | — | GitHub App credentials |

### Layer 2 — per-repo `.codent.yml`

Read from the **head** of the PR branch of the repo under review. Only the keys below are honoured;
unknown keys are ignored with a warning, and a malformed file falls back to env defaults rather than
failing the run.

```yaml
# .codent.yml — all keys optional
confidence_threshold: 65        # raise to reduce noise on a chatty repo
triviality_threshold: 40
rag_top_k: 8
exclude_paths:                  # never reviewed at all
  - "migrations/**"
  - "**/generated/**"
categories:                     # findings outside this list are suppressed
  - correctness
  - security
  - intent_mismatch
```

**Security note.** `.codent.yml` comes from an untrusted branch. It may only ever tune *numbers and
path globs* — it must never be able to specify a command, a path outside the repo, a model name, or
anything the sandbox executes. Any spec that adds a key breaking that rule is violating this section.

---

## 12. Design Decisions & Tradeoffs

### 12.1 — No LangGraph; plain async stages over a Pydantic state object

**Chosen because** the pipeline is a linear chain with exactly one conditional edge (Stage 6).
LangGraph earns its abstraction when there are cycles, checkpoint/resume, or human-in-the-loop
interrupts — none of which this pipeline has. Plain `async def stage(state, deps) -> state`
functions give the same composition with one less layer, and — decisively for a project built under
SDD — **each stage is a pure-ish function that a unit test can call directly with a fake `Deps` and
a hand-built `PipelineState`**, with no graph to construct and no framework to mock.

**Rejected:** LangGraph (carried over from `code-review-agent`) — costs an abstraction and a
dependency to model a control flow that is a straight line plus one `if`. **Rejected:** Celery/RQ
for orchestration — that is a *queue* concern (which we do have, at the job level), not a
*within-job* orchestration concern.

**Revisit if:** the pipeline grows genuine cycles (e.g. a repair loop that re-runs tests after the
model proposes a fix) or needs to resume mid-run after a crash. Both are real possibilities; neither
is v1.

### 12.2 — Claude behind an `LLMProvider` interface

**Chosen because** the entire product *is* the quality of the reasoning passes, and the architecture
already caps spend at ≤4 calls per PR by design. When you have deliberately made calls scarce, each
remaining call carries more weight and there is less budget to absorb a weak one — so the per-call
price is close to irrelevant next to the per-call quality. Claude's structured outputs and adaptive
thinking also map directly onto what Stages 1/4/5/6 need.

**Rejected:** Groq/LLaMA as the primary (carried over from `code-review-agent`) — free and fast, but
weakest exactly where the design has concentrated all its risk. It remains a legitimate *backend*
behind `LLMProvider` for cheap or bulk work, and the interface exists partly to keep that door open.

**Tradeoff accepted:** a paid API dependency and an outbound network requirement in the hot path.

### 12.3 — CodeBERT stays, behind an `Embedder` interface

**Chosen because** it is free, local, needs no API key, and the RAG layer's job (find me code that
looks like this hunk) is well within its ability. **Tradeoff:** a ~500 MB model and a slow cold
start, which will be felt on the first request after a deploy. The interface exists so that a hosted
code-embedding API can replace it later without touching `rag.py`'s callers.

### 12.4 — AST facts are *stated*, not *asked*

Carried over and formalized from `code-review-agent`'s `format_ast_context_for_llm`. The AST layer
computes facts deterministically and the prompt asserts them: "`process_batch` takes 2 positional
arguments; the call at line 88 passes 3." The model is never asked to derive something a parser
already knows. This is both a correctness decision (parsers do not hallucinate arity) and a token
decision (a stated fact is shorter than the code needed to derive it).

### 12.5 — No AI-authorship classifier

The proposal included one. **It was cut.** The pipeline treats every PR identically, and intent
reconstruction runs unconditionally (§4, call #1).

**Why:** the classifier's only job was to gate intent reconstruction, and gating it saves at most
one LLM call on some PRs while adding a whole heuristic subsystem — commit-pattern analysis, PR-body
phrasing, diff-uniformity scoring — that would need its own spec, its own tests, and its own
calibration against a labelled dataset nobody has. Worse, it puts a *guess* on the critical path:
a misclassified human PR silently loses intent-checking, and there is no signal that it happened.
Intent reconstruction is valuable on human PRs anyway — "does the diff do what the PR says it does"
is not an AI-specific question.

**Revisit if:** a use case appears that needs authorship as an *output* (e.g. a repo policy that
labels AI-authored PRs) rather than as an internal gate. That would be its own feature, not a
pipeline stage.

### 12.6 — Continuous risk score, not discrete severity labels

`code-review-agent` used `critical` / `warning` / `suggestion`. Those labels were an LLM output,
which means the noise level was a property of the prompt and could only be tuned by rewriting it.
A 0–100 score computed deterministically from four measurable channels moves the noise level into
`confidence_threshold` — a number a repo owner can change in a YAML file. The label the reviewer
sees is derived from the score, not the other way round.

### 12.7 — Pass B refutes; it does not re-review

Stage 6's prompt asks the model to argue the finding is *wrong*, and to default to `refuted` under
uncertainty. A symmetric "review it again" prompt tends to agree with the first pass (the finding is
in its context, phrased confidently), which makes the second call an expensive echo. Asymmetry is
what makes the 25-point `agreement` term meaningful.

### 12.8 — Batched calls, never per-file fan-out

See §4's batching rule. Stated twice on purpose: it is the failure mode most likely to creep in one
innocent spec at a time.

---

## 13. Deferred / Out of Scope

Recorded so they are decisions rather than oversights. Each becomes a spec if and when it is picked up.

| Item | Status | Note |
|---|---|---|
| Docker / Railway sandbox backend | **Deferred — required before public repos** | §10 promotion criterion |
| Languages other than Python | Deferred | Every language-specific layer is already behind an interface |
| Auto-fix / suggested-change commits | Out of scope for v1 | codent posts findings; humans push code |
| Repair loop (re-run tests after a proposed fix) | Out of scope | Would add cycles — see §12.1 revisit criterion |
| Learning from resolved/dismissed comments | Deferred | Needs a feedback store; a genuinely good v2 feature |
| Web dashboard | Out of scope | GitHub is the UI |
| AI-authorship classification | **Cut** | §12.5 |
| Multi-tenant billing / usage metering | Out of scope | |

---

## 14. Revisions

*Append-only. Do not edit the body of this document; add a revision block here instead.*

<!--
### Revision 1 — YYYY-MM-DD by <owner>
**Reason:** <what we learned>
**Change:** <what changed, in which section>
**Affected specs:** <spec ids that must be re-checked>
-->

---

**End of BUILD_SPEC.md.** Owners: keep this doc honest. If the code and this document disagree, one
of them is a bug — decide which, and fix that one.
