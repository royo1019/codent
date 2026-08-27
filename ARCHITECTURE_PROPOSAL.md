# codent — Architecture Proposal (draft, pending decision)

> ## ⚠️ SUPERSEDED — historical only
>
> The open questions in §3 were decided on 2026-08-27 and the result is **`BUILD_SPEC.md`**, which
> is now the architectural source-of-truth. Notable divergences from this draft: the AI-authorship
> classifier was **cut** (`BUILD_SPEC.md §12.5`), LangGraph was **dropped** in favour of plain async
> stages (`§12.1`), the LLM is **Claude** behind a provider interface (`§12.2`), and the sandbox is a
> **local subprocess runner** behind a `SandboxRunner` protocol until the Docker backend lands
> (`§10`).
>
> Keep this file for the decision history. **Never cite it as a source of truth; never edit it.**

> **Status:** Draft for review. Not yet approved. This is not the SDD `BUILD_SPEC.md` —
> it's the input Royo reads and decides on before that spec gets written.
>
> **Context.** `codent` is a from-scratch rebuild of the `code-review-agent` project
> (an AI PR-reviewer: GitHub App → webhook → LangGraph pipeline → static analysis (lint) +
> RAG (ChromaDB/CodeBERT) + AST (Python `ast`) → LLM review → inline GitHub comments).
> The rebuild's goal is a more ambitious, "complete product" version — with LLM call count
> minimized by design rather than retrofitted — built spec-by-spec under Spec-Driven
> Development with two owners: **royo** and **surya**.

---

## 1. Proposed pipeline

```
GitHub App (webhook) → Queue (per-repo, async)
        │
        ▼
   PR Intake
   ├─ AI-authorship classifier (heuristics: commit patterns, PR body phrasing, diff uniformity)
   └─ Intent reconstruction (LLM call #1 — reads PR description + linked issue + commits → spec)
        │
        ▼
   Repo Analysis (no LLM calls)
   ├─ AST index (symbol table, call graph, import graph) — built once per repo, cached
   └─ RAG index (per-repo vector collection, code embeddings) — built once per repo, cached
        │
        ▼
   Pre-filter Gate  ⟵ (optimization: reduce LLM calls)
   Per changed file: run linters + AST existence-check deterministically.
   If zero findings AND diff is trivial (below a line-change threshold) → skip LLM entirely,
   no comment. Only files that trip a signal proceed.
        │
        ▼
   Targeted Test Generation + Execution (LLM call #2, only for files that passed the gate)
   Generate tests aimed at the reconstructed intent's stated requirements; run in a sandbox;
   capture pass/fail + failure output as evidence.
        │
        ▼
   Reasoning Pass A (LLM call #3): given diff + AST facts + RAG neighbors + test result
   → is this a real issue?
        │
        ▼
   Adaptive Reasoning Pass B (LLM call #4 — CONDITIONAL, only when Pass A's signal is
   borderline; skipped when test failure + lint + AST already agree strongly) —
   independent corroboration/refutation
        │
        ▼
   Risk Scoring (deterministic, no LLM — combine: test result, pass A/B agreement,
   lint severity, AST signal into 0–100)
        │
        ▼
   Confidence Gate (deterministic threshold) → only findings ≥ threshold get posted
        │
        ▼
   GitHub posting (inline comments + summary + risk-scored verdict)
```

## 2. Key architecture decisions carried over or changed from `code-review-agent`

- **AST/RAG indexing pattern carries over unchanged.** Both are non-LLM, built once per
  repo, cached until the next push invalidates them. This was already correct in the old
  project (`rag.py`/`ast_parser.py`) and is the main reason neither indexing step costs an
  LLM call.
- **Pre-filter gate is a first-class pipeline stage from spec-01**, not a bolt-on retrofit.
  Trivial/clean files never reach the LLM.
- **AST existence-checks are promoted to a pre-computed "facts block"** the LLM is *told*,
  not asked to derive — formalizing what `format_ast_context_for_llm` did informally in the
  old project.
- **Test generation/execution is new infrastructure** — needs a sandboxed runner. This is a
  real security boundary (executing code from an AI-authored, possibly adversarial PR) and
  deserves its own spec with an explicit threat model, not a quick addition.
- **Reasoning Pass B is conditional, not automatic doubling.** This is the direct lever on
  LLM-call-count for the two-pass verification idea.
- **Risk scoring and confidence gating are pure deterministic logic** (no LLM call) sitting
  after the reasoning passes — replaces the old project's discrete
  `critical/warning/suggestion` severity labels with a continuous score plus a
  posting threshold.

## 3. Open questions (need Royo's decision before `BUILD_SPEC.md` is written)

- **Stack.** Keep the old project's stack (FastAPI, LangGraph, ChromaDB, CodeBERT via
  sentence-transformers, Groq/LLaMA) as-is, or reconsider any piece for the rebuild?
- **Sandboxed test execution.** What's the execution environment — subprocess with resource
  limits, Docker-per-job, a hosted sandbox service? This has real cost/latency/security
  tradeoffs and blocks the test-gen/execution spec from being written.
- **AI-authorship classifier.** Heuristic-only (commit metadata, PR body phrasing, diff
  uniformity), or should it also call the LLM for ambiguous cases? (Calling the LLM here
  would add to the call-count budget this whole redesign is trying to shrink.)
- **Intent reconstruction scope.** Should this run for every PR, or only for PRs the
  authorship classifier flags as AI-generated? Running it for every PR is one more
  guaranteed LLM call per PR; gating it narrows the win but may miss human PRs that still
  benefit from intent-checking.
- **Risk score formula.** What weights combine test result / pass-A-B agreement / lint
  severity / AST signal into the 0–100 score? Needs to be specified concretely before it's
  a testable unit.
- **Confidence threshold value(s).** A single global threshold, or per-repo/per-severity
  tunable? Where does it live (env var, per-repo config)?
- **Architecture-source-of-truth doc.** Since this is a fresh repo with no existing README,
  should `BUILD_SPEC.md` be authored fresh (this proposal becomes its seed), or should it
  start by porting relevant sections from the old project's `README.md` (module reference
  style, design-decisions-and-tradeoffs style) and then diverge?

---

**Next step once you've decided the above:** turn this into `BUILD_SPEC.md` (the permanent
architecture source-of-truth SDD points to), then write the generalized `SDD_PROCESS.md` +
`_template.md` + `specs/_tracker.csv` + `specs/royo/README.md` + `specs/surya/README.md`.
