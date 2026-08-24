---
name: research-to-package
description: Use when the user wants to "productionize", "port", "package", "harden", "clean up", or "make installable" research, academic, or prototype code — loose scripts, notebooks, or a one-off repo — turn scripts/notebooks into a real library, get something `pip`/`uv`-installable, or build an HPC/SLURM-friendly tool that runs one unit of work per invocation, even when they don't say the word "package". The core promise is faithfulness — the science/output is preserved exactly while structure, typing, tests, and docs are brought to production standard.
---

# Research → Production Package

Research code optimizes for getting a result once; production code optimizes for being run, trusted, and extended by others. The gap between them is mostly packaging, structure, typing, error handling, tests, and docs — NOT the algorithm. This skill closes that gap **without changing what the code computes.**

## The one rule that governs everything: faithfulness first

The original output is the spec. A hardened package that produces different numbers is a regression, no matter how clean it is. So every behavior-changing improvement (performance work, vectorization, library upgrades) is quarantined behind a regression test and explicit user approval. Structural and stylistic quality is always-on; anything that could move the output is opt-in and gated.

**Faithfulness means scientific/data outputs and existing user-visible semantics.** Operational packaging behavior — import location, package metadata, logging plumbing, temporary-file routing, and an explicitly approved tool wrapper — may change where this skill requires it, provided those changes do not alter scientific/data results. Any intentional operational-interface change must be identified in Phase 0 and recorded in `CHANGELOG.md`.

When you cannot tell whether a change affects output, treat it as if it does.

**Violating the letter of these gates is violating their spirit.** "Technically the output didn't change" is not a defense if you skipped the re-run that would have proven it, and "I was following the intent" is not a defense for skipping a STOP. The checks ARE the intent.

**The environment is part of the spec.** A library upgrade can move numeric output as surely as a code edit. Capture the original's *resolved* dependency versions (lockfile / installed env) and reproduce them in an **isolated virtualenv** before porting — never install into a shared/base/conda env where `pip install -e .` can silently upgrade `numpy`/`numba`/etc. out from under the baseline. Pin to the original's versions; upgrading them is Phase 4 work.

**Re-verify after every behavior-touching edit, in any phase** — not just Phase 4. The regression baseline only guards anything if you actually re-run it. Any refactor, dead-code removal, or "harmless" cleanup re-runs the baseline before you call it done.

## Work in phases, and STOP at every gate

Hardening a whole repo in one pass is how agents go off the rails — silent rewrites, scope creep, broken science. Decompose into phases with hard stops. At each STOP, summarize what you did and **wait for the user's explicit "approved" before continuing.** Never skip a gate, even if the next phase seems obvious.

```
PHASE 0  Recon (read-only)        → plan, then STOP
PHASE 1  Scaffold package         → install works, then STOP
PHASE 2  Faithful port + quality  → diff summary, then STOP
PHASE 3  Tests + docs             → reports green, then STOP
PHASE 4  Optimize (OPT-IN only)   → per-change diffs, gated
```

The pipeline assumes "messy scripts," but real inputs are often already partway to a package. **Grade the source and mark phases that are already satisfied as N/A** (see Phase 0) — a typed, tested module needs a structural touch-up, not a five-phase rewrite. You may skip an N/A phase after stating why in Phase 0, but never merge or bypass the STOP for any phase in which work is actually performed.

### Phase 0 — Recon (original source read-only)
Read the source as read-only reference. Trace the real entrypoint from input to output. Produce:
- **Source maturity grade** — script(s) → module → partial package → package. Grade it honestly and say which phases you are collapsing or skipping as a result.
- **Shape: library or tool?** — a *library* (the importable API is the product, e.g. a scikit-learn estimator, a parser, a model class) vs a *tool* (file-in/file-out, one unit of work per invocation). This decides whether the Execution Contract and `cli.py` apply **at all**. A CLI bolted onto a library is dead scaffolding. When unclear, ask.
- **Names** — the import name, the distribution (pip) name, and the repo/remote name can all legitimately differ; pin each now. The public import name is a decision, not a default.
- the runtime dependency list **with the original's resolved versions** (lockfile or installed env) — that environment is part of the spec,
- a file-by-file port plan mapping old → new locations,
- every hardcoded path, global, and assumption,
- all temp-file / scratch usage,
- code that is plausibly dead (mark it; do not delete yet),
- hot loops that are candidates for the optional Phase 4,
- a minimal representative regression fixture and the original output produced from it, stored outside the read-only source tree. If no suitable input exists, create a small synthetic fixture outside the source tree and run the original on it. If the original cannot run, STOP and obtain user-approved golden values before any behavior-touching port begins. Phase 3 turns this baseline into the committed regression test.

STOP and present the plan. Resolve ambiguities with the user here, before writing any code.

### Phase 1 — Scaffold
Create the package skeleton (see Architecture) with stubs and a `pyproject.toml`, **in an isolated virtualenv pinned to the original's resolved dependency versions**. Do not migrate logic yet. STOP once `uv pip install -e .` (into that venv) succeeds and the package imports. If the install upgraded any runtime dependency, flag it — that is a faithfulness risk, not a detail.

### Phase 2 — Faithful port + always-on quality
Move the logic into the new structure, applying the **always-on** quality standards below. **For a *tool***, wire the execution contract (CLI, config, logging, exit codes, tmp-dir routing) only to the extent explicitly agreed in Phase 0. **For a *library***, skip all of that — the importable API is the entire surface. Scientific/data outputs and existing user-visible semantics must match the original, except for operational-interface changes explicitly approved in Phase 0. Re-run the Phase 0 regression baseline after every behavior-touching edit. STOP with a diff summary mapping what moved where (this seeds `CHANGELOG.md` — see Phase 3).

### Phase 3 — Tests + docs
Add a small synthetic fixture (build it in code; commit no large data). Write unit tests for the core, a CLI smoke test *if it is a tool*, and — critically — a **regression test** asserting the new output matches the original baseline on the fixture. Use **exact equality for discrete/structured output** (labels, indices, JSON, arrays of ints); **explicit tolerance only for floats**. Formalize the Phase 0 baseline as the committed regression test. If the baseline could not be captured in Phase 0, the user must supply or approve golden values before any behavior-touching work proceeds — never invent a baseline. Add Sphinx-autodoc-compatible docstrings and a README covering import usage, CLI usage *if a tool*, and (if relevant) a batch/SLURM snippet. Write a **CHANGELOG.md** that records every change from the original — grouped by phase — and, for each, *why* it was made (faithfulness-neutral cleanup, structural fix, documented strict-mode relaxation, gated Phase 4 optimization). Its audience is the original researcher: explain the benefit of each refactor so they can see the value and learn from it, rather than just listing diffs. Run the linter, the type checker (strict — see relaxations note below), and the tests; report results. STOP.

### Phase 4 — Optimize (opt-in, output-risking)
Run ONLY if the user explicitly asks. Profile first; touch only proven hot loops. Library/dependency upgrades belong here, not earlier. Each change must keep the Phase 3 regression test green (exact, or within tolerance for floats) and be presented as a before/after diff with the measured speedup. Any change that pushes output past the baseline is reverted, not committed.

## Architecture (src layout)

A `src/` layout prevents accidental imports of the working tree and is the modern packaging default. Adapt names to the domain — the shape is what matters. Items marked `[TOOL]` exist only for the *tool* shape; omit them for a library.

```
src/<pkg>/
  __init__.py     # public API surface ONLY: the entrypoint fn(s)/class(es) + __version__
  py.typed        # PEP 561 marker — makes your type hints count for downstream consumers
  core/           # the ported logic, as typed, mostly-pure functions
  errors.py       # custom exception types
  cli.py          # [TOOL] thin arg-parsing layer → calls core. NO business logic here.
  config.py       # [TOOL] frozen dataclass of run parameters (input, output_dir, tmp_dir, threads, ...)
  io/             # [TOOL] reading/validating inputs, writing outputs
  logging.py      # [TOOL] logger setup (to stderr)
tests/            # unit + regression test (+ 1 CLI smoke for a tool)
pyproject.toml    # PEP 621, modern build backend (+ console_scripts entry point for a tool)
README.md
CHANGELOG.md      # the porting record: what changed vs the original, grouped by phase, and WHY
```

The import name in `src/<pkg>/` need not equal the distribution name in `pyproject.toml` `[project].name`, nor the repo/remote — set each deliberately (see Phase 0).

**For a tool**, two entrypoints share one implementation: the CLI and the Python API both call the same `core` function so they can never drift.
```
CLI:    <cmd> --input X --output-dir out/ [--tmp-dir P] [--threads N]
Python: from <pkg> import run; run(input, output_dir, tmp_dir=None, threads=1, ...)
```
**For a library**, there is one entrypoint — the importable API — and no `cli.py`/`config.py`/`io/`/`logging.py`.

## Execution contract (batch / HPC friendly) — TOOLS ONLY

**Skip this entire section for a library.** Its product is the importable API; a CLI / `--threads` / `--tmp-dir` contract would be artificial. For a *tool*, even when the user doesn't say "SLURM", these defaults make it composable and safe to run at scale, so apply them unless they conflict with the original design:

- **One unit of work per invocation** (one file / sample / dataset). No hidden batch mode.
- **Threads explicit** via `--threads` (default 1). No implicit parallelism.
- **Outputs** go under a caller-supplied `--output-dir`. No hardcoded paths.
- **Optional `--tmp-dir`** for scratch (e.g. node-local fast disk). Default to `$TMPDIR` if set, else the system temp dir. Route ALL intermediates there via `tempfile(dir=...)`, and always clean up on exit — including on failure.
- **Exit code 0 on success, non-zero on any failure.** No interactive prompts.
- **Logs/progress to stderr**; results/data to files or stdout only.
- **Deterministic** given the same input + args. Add no new runtime network dependency. If the original inherently requires network access, preserve and flag it in Phase 0 rather than silently removing it.

## Code quality: two tiers

Hold the work to a high bar, but remember that SOLID + "extensibility" is exactly what tempts an agent to over-abstract a simple tool. Keep the two tiers separate in your head.

**ALWAYS-ON (behavior-neutral — apply during the port):**
- *Single Responsibility / Curly's Law*: each module and function does one thing.
- *DRY*: factor out genuine duplication only — not coincidental similarity (and not numerically-identical-but-clearer code where factoring risks float drift).
- *Dependency Inversion*: pass collaborators/config in; no global state or hidden singletons.
- *Interface Segregation*: small focused functions over fat utility modules.
- High cohesion, low coupling; keep `core/` functions pure where possible.
- Full type hints on every signature; pass the type checker in **strict** mode — *minus a documented, principled set of ecosystem relaxations* where the stack makes literal strict impractical (numpy's `ndarray` generics, libraries that ship no `py.typed` such as scikit-learn, untyped decorators such as numba's `@njit`). Relax those specific flags in config **with a comment**, and localize each `# type: ignore` / `cast` to the genuine untyped seam — never blanket-disable strict.
- Ship a `py.typed` marker so consumers actually get your hints.
- Sphinx-autodoc-compatible docstrings (napoleon/Google or reST) on every public symbol.
- Structured error handling: raise typed exceptions from `errors.py`; no bare `except`. If these exceptions are part of the public surface, document them (and have them subclass the relevant built-in so existing `except ValueError` handlers keep working).
- Readable internal names — but do NOT rename the **public API, CLI surface, or output filenames**; those are a contract.
- Remove provably-dead code (unreferenced, unreachable) **as its own flagged decision, surfaced at a STOP — not silently bundled into the port** — and delete only after the baseline re-runs green. When unsure, keep it and flag it.

**BOUNDED (anti-over-engineering — this is usually a small tool):**
- Minimal viable abstraction. YAGNI. No speculative base classes, plugin systems, or registries "for extensibility."
- Prefer functions to classes unless state genuinely demands a class.
- Open/Closed and Liskov apply ONLY where real polymorphism already exists in the original.

## Forbidden (Phases 0–3)
- Modifying or deleting the **original source files** — they remain read-only reference. New package files may be created in the agreed target directory/worktree. Port logic by copying/refactoring from the original; do not destructively move the originals.
- Adding dependencies beyond the original's (plus the test/lint/type toolchain) without asking.
- Upgrading the original's dependency versions (that is Phase 4) — pin to what it resolved.
- Adding features the original lacks: batch mode, config-file loaders, plotting, web/API, Docker, or CI. The one exception is the thin execution wrapper explicitly agreed in Phase 0 for a *tool* shape; it may expose existing functionality but must not add scientific capability.
- Adding a CLI / execution contract to a *library* shape, or adding one to a *tool* unless the tool shape and wrapper surface were explicitly agreed in Phase 0.
- Changing algorithm logic, constants, or output formats.
- Optimizing or vectorizing (that is Phase 4).
- Writing intermediates outside the resolved tmp-dir, or leaving scratch behind.

## Stop and ask before
- Adding any dependency not in the original, or changing a resolved version.
- Any change that could alter numerical/output behavior (defer to Phase 4).
- Deleting code you cannot prove is dead.
- Deleting or overwriting anything outside the new package directory.

## Red flags — STOP and re-check
You are about to break faithfulness or skip a gate if you catch yourself thinking:
- "The next phase is obvious, I'll skip this STOP."
- "This cleanup is behavior-neutral — no need to re-run the baseline."
- "The diff is trivial / the user clearly wants it — I'll collapse the phases."
- "I can tell this refactor doesn't move the output" (without running it).
- "Upgrading this library is harmless and cleaner."
- "This code is clearly dead, I'll just delete it."
- "I'll add a small config loader / CLI flag / base class while I'm here."

Every one of these means: stop, re-run the regression baseline, and present at the gate before continuing.

## Rationalizations — and the reality

| Excuse | Reality |
|--------|---------|
| "Obviously behavior-neutral — skip the baseline re-run." | "Obviously" is how output drift ships. The baseline guards nothing unless you re-run it — every time, in every phase. |
| "The diff is trivial, I'll skip this STOP." | Gates aren't sized to the diff; they're where the user catches scope creep, renamed outputs, and silent version bumps. STOP anyway. |
| "The user wants it done, so I'll collapse the phases." | Only Phase 0 may declare later phases N/A; phases containing actual work retain their individual STOPs. Speed is not approval. |
| "I can tell this refactor won't change the output." | If you could tell without running it, you wouldn't need a regression test. When unsure, treat it as behavior-changing. |
| "Upgrading numpy/this lib is harmless and cleaner." | A version bump moves numbers like a code edit can. That is gated Phase 4 work — never a silent Phase 2 detail. |
| "This code is clearly dead, I'll just delete it." | Prove it (unreferenced and unreachable), flag it at a STOP, and delete only after the baseline re-runs green. When unsure, keep it. |
| "I'll add a tiny config loader / CLI / base class while I'm here." | Adding features the original lacks is forbidden in Phases 0–3. YAGNI. The original is the spec. |
