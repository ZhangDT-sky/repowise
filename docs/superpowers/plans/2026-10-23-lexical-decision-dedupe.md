# Lexical Decision Dedupe Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add an explicit `repowise decision dedupe --lexical` mode that folds reworded decisions sharing an evidence commit on mock indexes.

**Architecture:** Keep the existing semantic planner and fold implementation intact. Add a core lexical planner that reuses `_foldable`, `_star_cover`, and `DedupePlan`, comparing `decision_match_text` with normalized `difflib.SequenceMatcher` ratios only within shared evidence-commit groups. The CLI selects lexical or semantic planning; lexical mode bypasses embedder/vector-store setup while semantic mode preserves the current mock refusal.

**Tech Stack:** Python 3.12, Click, SQLAlchemy async sessions, pytest/pytest-asyncio.

---

### Task 1: Add the lexical planner regression tests

**Files:**
- Modify: `tests/unit/persistence/test_decision_dedupe.py`

- [x] **Step 1: Add a test for same-commit reworded matches.**

Use the existing `_seed` helper and assert `plan_lexical_dedupe` returns one folded pair when two `pr` records share `commits=["sha-1"]` and have the issue's reworded title/decision text.

- [x] **Step 2: Add tests for provenance and threshold boundaries.**

Cover similar text with different commits, missing commits, and a pair below the configured threshold. Each must produce no lexical cluster.

- [x] **Step 3: Add an apply/idempotence test.**

Run `apply_dedupe(..., lexical=True)` or the selected lexical apply API, assert evidence and alias behavior match the semantic fold, then run it again and assert `folded_count == 0`.

- [x] **Step 4: Run the new tests and verify they fail for the missing planner.**

Run:

```powershell
pytest tests/unit/persistence/test_decision_dedupe.py -k lexical -q
```

Expected: collection or attribute failures because the lexical planner/API does not exist yet.

### Task 2: Implement lexical planning in the core dedupe module

**Files:**
- Modify: `packages/core/src/repowise/core/persistence/decision_dedupe.py`
- Modify: `packages/core/src/repowise/core/analysis/decisions/semantic_match.py` only if a small shared normalization helper is needed

- [x] **Step 1: Add deterministic lexical helpers.**

Import `difflib`, parse `evidence_commits_json` with the existing JSON helper, normalize whitespace/case, and compute `SequenceMatcher(None, normalized_left, normalized_right).ratio()` over `decision_match_text`.

- [x] **Step 2: Build same-commit candidate edges.**

For foldable records, map each evidence commit to record ids. For each commit group, compare each unordered pair once, keep the maximum score for a pair, and ignore records with no commit. Use the existing `DedupePlan(tau=...)`, `plan.eligible`, `plan.pairs`, and `_star_cover` so canonical selection and cluster semantics stay unchanged.

- [x] **Step 3: Expose lexical planning without weakening semantic planning.**

Add `plan_lexical_dedupe(session, repository_id, tau=...)` to `__all__`. Keep `plan_dedupe`'s vector-store gate unchanged. If apply needs to know the plan is lexical, pass the normal `DedupePlan`; `apply_dedupe` must continue using the existing fold path and must not query vectors when the lexical caller supplies a plan.

- [x] **Step 4: Run the persistence tests.**

Run:

```powershell
pytest tests/unit/persistence/test_decision_dedupe.py -q
```

Expected: all existing semantic tests and the new lexical tests pass.

### Task 3: Add the explicit CLI flag and branch

**Files:**
- Modify: `packages/cli/src/repowise/cli/commands/decision_review_cmd.py`
- Modify: `tests/unit/cli/test_decision_cmd.py` or the repository's decision review CLI test module selected after locating command tests

- [x] **Step 1: Add a CLI regression test for lexical mode.**

Invoke `decision dedupe --lexical` against a mock-configured repository with monkeypatched session/open helpers. Assert embedder and vector-store builders are not called, and the command reaches the lexical planner.

- [x] **Step 2: Add a regression test preserving the default mock refusal.**

Invoke the command without `--lexical` and assert the existing `no_real_embedder` refusal remains.

- [x] **Step 3: Add the `--lexical` Click option and route execution.**

Extend `dedupe_command` with `lexical: bool`. In lexical mode skip `build_embedder`, `build_vector_store`, and vector-store cleanup; call `plan_lexical_dedupe`. In semantic mode retain the current code and messages. Reuse `apply_dedupe` with the supplied plan and keep JSON/report rendering unchanged.

- [x] **Step 4: Run focused CLI tests.**

Run the new CLI test selection plus existing decision command tests:

```powershell
pytest tests/unit/cli/test_decision_cmd.py -q
```

Expected: all pass, including the default mock refusal.

### Task 4: Refine docs/help and run the complete verification set

**Files:**
- Modify: `packages/cli/src/repowise/cli/commands/decision_review_cmd.py` if help text needs clarification
- Modify: `tests/unit/persistence/test_decision_dedupe.py` and CLI tests only for failures found during verification

- [x] **Step 1: Verify command help and output semantics.**

Check `repowise decision dedupe --help`; it must describe lexical mode and that `--tau` is the similarity threshold for the selected mode.

- [x] **Step 2: Run the full affected test set.**

```powershell
pytest tests/unit/persistence/test_decision_dedupe.py tests/unit/cli/test_decision_cmd.py -q
```

- [x] **Step 3: Run static checks used by the repository.**

Use the project-configured formatter/linter/type checks discovered from `pyproject.toml`; at minimum run the targeted Python compile/import check if no narrower command is configured.

- [x] **Step 4: Review the diff and commit implementation.**

Confirm no unrelated files changed, then commit with:

```powershell
git add packages/core/src/repowise/core/persistence/decision_dedupe.py packages/cli/src/repowise/cli/commands/decision_review_cmd.py tests/unit/persistence/test_decision_dedupe.py tests/unit/cli/test_decision_cmd.py
git commit -m "feat: add lexical decision dedupe mode"
```
