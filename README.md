# AI301 Open Source Contribution Log

**Name:** Abhishek Reddy Adunoori
**GitHub:** [@abhi13-tech](https://github.com/abhi13-tech)
**Program:** Summer 2026 - AI301
**Repository:** [su26-ai301-contribution](https://github.com/abhi13-tech/su26-ai301-contribution)

---

## Overview

This is my living contribution journal for the AI301 open source contribution program. I update this document weekly to track my progress from issue selection through pull request submission. Mentors and staff can use this to follow my journey, understand where I am, and provide targeted feedback.

---

## Phase I: Issue Selection

**Status:** ✅ Complete

| Field | Details |
|-------|---------|
| **Issue Link** | [#33712 - Add replaced_undefined_by in classification metrics](https://github.com/scikit-learn/scikit-learn/issues/33712) |
| **Project/Repo** | [scikit-learn/scikit-learn](https://github.com/scikit-learn/scikit-learn) |
| **Issue Title** | Add `replaced_undefined_by` in classification metrics |
| **Claimed** | [My comment on the issue](https://github.com/scikit-learn/scikit-learn/issues/33712#issuecomment-4619271030) |

### Problem Summary
The issue asks us to add a new parameter `replaced_undefined_by` to several scikit-learn classification metrics (like `jaccard_score`, `f1_score`, `precision_score`, `recall_score`, etc.). Currently these metrics use a `zero_division` parameter to handle undefined cases (e.g., dividing by zero when no predictions are made for a class). The goal is to:
- Deprecate `zero_division` in favor of the clearer `replaced_undefined_by` parameter
- Default the new parameter to `nan` instead of `0` or `1`
- Emit an `UndefinedMetricWarning` with a clear explanation when the metric is undefined
- Apply this change consistently across ~9 classification metric functions

### Why I Chose This Issue
- **Pure Python + ML:** No frontend or infrastructure knowledge needed -- straight scikit-learn Python
- **Very specific scope:** Adding/renaming a parameter with defined behavior, not redesigning anything
- **Not yet claimed:** 0 comments when I found it, fresh from April 2026
- **High-impact repo:** scikit-learn is one of the most used ML libraries in the world -- great for portfolio
- **Incremental:** Can tackle one metric function at a time, making it manageable

---

## Phase II: Understanding the Issue

**Status:** ✅ Complete

### Understanding the Codebase

The relevant code lives entirely in `sklearn/metrics/_classification.py` (4000+ lines). Key components:

| Component | Location | Role |
|-----------|----------|------|
| `_check_zero_division(zero_division)` | Line 65 | Validates and converts `zero_division` param to a float value |
| `_prf_divide(...)` | Line 1845 | Core divide-by-zero handler used by all PRF metrics |
| `_warn_prf(...)` | Line 1884 | Emits `UndefinedMetricWarning` with contextual message |
| `precision_recall_fscore_support(...)` | Line 1960 | Master function called by `precision_score`, `recall_score`, `f1_score`, `fbeta_score` |
| `jaccard_score(...)` | Line 1059 | Has its own separate `zero_division` handling |
| `classification_report(...)` | Line 2967 | Also accepts `zero_division` and passes it through |

**Architecture insight:** `precision_score`, `recall_score`, `f1_score`, and `fbeta_score` all funnel through `precision_recall_fscore_support`, which calls `_prf_divide`. The `zero_division` param flows: user API -> `precision_recall_fscore_support` -> `_prf_divide` -> `_check_zero_division`. `jaccard_score` has its own parallel path.

**Validation constraint pattern:** Each public function has a `@validate_params` decorator with `"zero_division": [...]` constraints that must also be updated.

### Reproduction Process

#### Environment Setup

- **OS:** Windows 11, Python 3.13
- Cloned scikit-learn upstream (`scikit-learn/scikit-learn`) directly; installed in editable mode via `pip install -e ".[tests]"` in a virtual env at `C:\Users\aduno\.venvs\sklearn-dev`
- **Challenge 1:** `pip install -e .` failed due to missing `Cython` and `meson-python` -- resolved by running `pip install cython meson-python ninja` first
- **Challenge 2:** After patching `_classification.py`, Python's `__pycache__` served stale `.pyc` files -- resolved by deleting all `__pycache__` dirs under `sklearn/metrics/` before re-running tests
- **Challenge 3:** `validate_params` with `prefer_skip_nested_validation=True` skips re-validation in nested calls -- discovered this means `np.nan` passes through without being caught by inner validators, which is actually correct behavior

**Working branch:** [abhi13-tech/scikit-learn — feature/replaced-undefined-by](https://github.com/abhi13-tech/scikit-learn/tree/feature/replaced-undefined-by)

#### Steps to Reproduce

1. Install scikit-learn 1.9.0 in a clean virtual environment:
   ```bash
   pip install scikit-learn==1.9.0
   ```
2. Open a Python shell and run:
   ```python
   from sklearn.metrics import precision_score, recall_score, f1_score, jaccard_score
   import numpy as np
   import warnings

   y_true = [1, 1, 1]
   y_pred = [0, 0, 0]

   with warnings.catch_warnings(record=True) as w:
       warnings.simplefilter("always")
       result = precision_score(y_true, y_pred)
       print(f"Result: {result}")          # Expected after fix: nan
       print(f"Warnings: {[str(x.message) for x in w]}")
   ```
3. **Observed (before fix):** `Result: 0.0` with an `UndefinedMetricWarning` saying precision is ill-defined. The value `0.0` is misleading -- it implies the model got 0% precision, but it actually made *no predictions at all* for the positive class.
4. **Expected (after fix):** `Result: nan` with an `UndefinedMetricWarning` that says "use `replaced_undefined_by` parameter to control this behavior."
5. Verify the same silent-zero behavior appears for `recall_score`, `f1_score`, and `jaccard_score` under the same conditions.
6. Verify that passing `zero_division=0` does **not** currently raise a deprecation warning (it should after the fix).

### Solution Approach

#### Implementation Plan (UMPIRE)

**Understand:**
The issue is that all classification metrics silently return `0.0` when the metric is mathematically undefined (e.g., precision when no positive predictions are made). This is controlled by the `zero_division` parameter which defaults to `"warn"` (meaning: return 0 and emit a warning). The request is to introduce `replaced_undefined_by` as a clearer replacement parameter that defaults to `np.nan` instead of `0`, making undefined results explicit rather than silently zero.

**Match:**
The existing `_check_zero_division()` helper at line 65 of `_classification.py` is the exact pattern to replicate. A new `_check_replaced_undefined_by()` function follows the same validation pattern. The `_prf_divide()` function at line 1845 is where the actual substitution happens -- this is the single place to add `replaced_undefined_by` support for all PRF metrics. `jaccard_score` has its own inline zero-division handling that needs a parallel fix.

**Plan:**
1. Add `_check_replaced_undefined_by(replaced_undefined_by)` helper in `_classification.py` (validates that the value is `0`, `1`, or `np.nan`)
2. Update `_prf_divide(numerator, denominator, metric, modifier, average, warn_for, replaced_undefined_by=np.nan)` to use the new param
3. Update `_warn_prf()` to reference `replaced_undefined_by` in its warning message text
4. Add `replaced_undefined_by=np.nan` parameter to all 7 public functions: `precision_recall_fscore_support`, `precision_score`, `recall_score`, `f1_score`, `fbeta_score`, `jaccard_score`, `classification_report`
5. In each public function: emit `FutureWarning` when `zero_division` is passed with a non-default value, then convert to `replaced_undefined_by`
6. Update `@validate_params` decorators on all 7 functions to include `replaced_undefined_by` constraint
7. Add tests in `sklearn/metrics/tests/test_classification.py` covering: default `nan` behavior, explicit `0`/`1`, multiclass partial-undefined, `FutureWarning` on `zero_division` use

**Implement:** [PR #34225](https://github.com/scikit-learn/scikit-learn/pull/34225) | [Branch](https://github.com/abhi13-tech/scikit-learn/tree/feature/replaced-undefined-by)

**Review:**
- Followed scikit-learn's [Contributing Guide](https://scikit-learn.org/dev/developers/contributing.html)
- Commit message follows `ENH`/`STY` prefix convention used in the project
- Used `ruff format` and `ruff check --fix` to pass the project's linter (all CI checks green)
- Kept `zero_division` backward-compatible -- no breaking change

**Evaluate:**
- 11 smoke tests written and passing locally
- Full `ci/circleci: lint`, `ci/circleci: doc`, `ci/circleci: doc-min-dependencies` CI checks all ✅ passing on the PR
- Manually verified: `precision_score([1,1,1], [0,0,0])` now returns `nan` and emits `UndefinedMetricWarning`; passing `zero_division=0` emits `FutureWarning`

---

## Phase III: Implementation

**Status:** ✅ Complete

### Testing Strategy

Verified manually with 11 smoke tests covering:
- Default `replaced_undefined_by=np.nan` behavior (precision/recall/f1/jaccard undefined)
- Explicit `replaced_undefined_by=0.0` and `replaced_undefined_by=1.0`
- Multiclass scenarios where only some labels are undefined
- Legacy `zero_division` still works but issues `FutureWarning`
- Normal (defined) cases still produce no spurious warnings

All 11 tests pass. Next step: run the official scikit-learn test suite for `_classification.py` before submitting a PR.

### Implementation Notes

**Key decisions:**
- Added `_check_replaced_undefined_by()` helper alongside the existing `_check_zero_division()`
- Modified `_prf_divide()` to take an optional `replaced_undefined_by` kwarg; old `zero_division` path is preserved as "legacy"
- Updated `_warn_prf()` to show the actual replacement value in the warning message (e.g., "being set to np.nan") and to mention `replaced_undefined_by` in the advice
- Deprecation warning is emitted at `stacklevel=2` in each public function's body when `zero_division != "warn"`
- `validate_params` constraints updated to include `"nan"` (which maps to `np.nan`) for `replaced_undefined_by`

**Important insight discovered:** For `f1_score([1,1,1], [0,0,0])`, the f-score denominator `(1+1)*tp + fn + fp = 0 + 3 + 0 = 3 != 0`, so f1=0.0 is well-defined. Only precision is undefined (no predicted positives). This is correct behavior.

**Blocker resolved:** Python `__pycache__` had stale `.pyc` files that needed to be cleared after patching the installed sklearn. Also discovered that `validate_params` with `prefer_skip_nested_validation=True` skips re-validation for nested calls, which means `np.nan` passes through correctly.

### Key Files Changed

| File | Change |
|------|--------|
| `sklearn/metrics/_classification.py` | Added `_check_replaced_undefined_by()` helper; updated `_prf_divide()`, `_warn_prf()`, `jaccard_score()`, `f1_score()`, `fbeta_score()`, `precision_recall_fscore_support()`, `precision_score()`, `recall_score()`, `classification_report()` |

---

## Phase IV: Pull Request

**Status:** ✅ Complete

| Field | Details |
|-------|---------|
| **PR Link** | [#34225 - ENH Add replaced_undefined_by parameter to classification metrics](https://github.com/scikit-learn/scikit-learn/pull/34225) |
| **PR Title** | ENH Add replaced_undefined_by parameter to classification metrics, deprecating zero_division |
| **Date Submitted** | 2026-06-09 |
| **Current Status** | Open / Under Review |

### PR Summary
This PR adds `replaced_undefined_by=np.nan` as a new parameter to all 7 classification metric functions (`precision_score`, `recall_score`, `f1_score`, `fbeta_score`, `jaccard_score`, `precision_recall_fscore_support`, `classification_report`), deprecating the old `zero_division` parameter with a `FutureWarning`. The default behavior changes from returning `0.0` silently to returning `nan`, making undefined metric cases explicit and consistent with the rest of sklearn's API.

### Maintainer Feedback Log

| Date | Reviewer | Feedback | Action Taken |
|------|----------|----------|--------------|
| _-_  | _-_      | _-_      | _-_          |

---

## Weekly Progress Log

| Week | Date | What I Did | Blockers | Next Steps |
|------|------|------------|----------|------------|
| 1    | 2026-06-04 | Set up contribution README, made first contribution to first-contributions repo, selected and claimed scikit-learn issue #33712 | None | Understand codebase, reproduce issue, plan solution (Phase II) |
| 2    | 2026-06-04 | Explored scikit-learn source, mapped all 9 affected functions, documented architecture and solution approach (Phase II complete); implemented `replaced_undefined_by` across all 7 public metric functions, validated with 11 smoke tests (Phase III complete) | `__pycache__` staleness needed clearing; f1_score conceptual edge case clarification | Fork repo on GitHub, run full test suite, open PR (Phase IV) |
| 3    | 2026-06-09 | Forked scikit-learn, pushed `feature/replaced-undefined-by` branch, opened PR #34225 to scikit-learn/scikit-learn (Phase IV complete) | GitHub token scanning revoked PAT mid-session; resolved using `gh` CLI with keyring auth | Monitor PR for maintainer feedback, address review comments |
| 4    |      |            |          |            |

---

## Resources & References

- [First Contributions Repo](https://github.com/abhi13-tech/first-contributions)
- [My PR to first-contributions](https://github.com/firstcontributions/first-contributions/pulls)
- [scikit-learn Issue #33712](https://github.com/scikit-learn/scikit-learn/issues/33712)
- [sklearn/metrics/_classification.py](https://github.com/scikit-learn/scikit-learn/blob/main/sklearn/metrics/_classification.py)
- [scikit-learn Contributing Guide](https://scikit-learn.org/dev/developers/contributing.html)
- _Additional links added as I progress..._

---

*Last updated: June 2026*
