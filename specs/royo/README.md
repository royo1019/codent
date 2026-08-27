# royo — spec queue

Specs in this folder are **owned by royo**. Ownership means royo scoped them, approved them, reviews
the unit summary blocks, and signs off "ready to ship". Claude executes; royo decides.

Read `../../BUILD_SPEC.md` once per session for architecture. Read `../../SDD_PROCESS.md` every time
you do feature work. The live status of every unit is in `../_tracker.csv` — that CSV is the index,
not this file.

---

## Queue

| Spec | Slug | Status | Notes |
|---|---|---|---|
| _(none yet)_ | — | — | — |

*Add a row when a spec is drafted; update it when it's approved and when it completes. Unit-level
detail belongs in the tracker, not here.*

---

## How to prompt Claude

### Start a new spec

```
Read BUILD_SPEC.md and SDD_PROCESS.md.
Owner: royo
Feature: <one paragraph describing what and why>

Enter Spec Authoring Mode. Interrogate me on anything ambiguous,
then produce a draft spec at specs/royo/<your-suggested-slug>/spec.md
and add unit rows to specs/_tracker.csv. Run the Goldfish Gate before
handing it back. No code yet.
```

### Approve and start building

```
The draft at specs/royo/<feature-slug>/spec.md is approved.
Flip the tracker rows for this spec from spec_drafted to not_started,
append the approval line to BUILD_LOG.md, then enter Build Mode for
the first unit. Stop after the first unit and emit the unit summary
block.
```

### Just continue my work (the default once a spec is in flight)

```
Read SDD_PROCESS.md. I'm royo. Find my next pending unit across
specs/royo/ and specs/_tracker.csv per §9.6. Tell me in 2–3 sentences
what you're about to build (Goldfish check), then enter Build Mode for
that one unit. Stop after the unit summary block.
```

### Status only

```
Read specs/_tracker.csv. I'm royo. Per §9.6, give me my queue: per
spec, units done / in_progress / not_started / blocked, and which spec
is closest to shipping. Cite row numbers.
```

Full template set — including mid-build revisions and continuous build — is in
`../../SDD_PROCESS.md §15`.

---

## Session wrap-around

```bash
# start
git checkout main && git pull
git checkout -b royo/<feature-slug>        # or: git checkout royo/<feature-slug> && git pull --rebase origin main

# ... run Claude Code ...

# end
git status
git add specs/ src/ tests/
git commit -m "royo/<feature-slug>: <unit_id> — <one-line summary>"
git push -u origin royo/<feature-slug>
```

---

## Before approving a spec, check

- **§3a is fully answered** and no row is a violation.
- **The LLM call budget** (`BUILD_SPEC.md §4`) still holds — no fifth unconditional call, no
  per-file fan-out.
- **`Files touched` collisions** with surya's in-flight specs, especially on the three shared paths:
  `src/codent/pipeline/runner.py`, `src/codent/main.py`, `src/codent/models/state.py`.
- **Every unit is atomic** — none depends on a later one.
- **The wiring test asserts `llm_calls_made`** if it runs the pipeline end-to-end.
