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

Current behavior (scikit-learn 1.9.0):
```python
from sklearn.metrics import f1_score
import numpy as np

y_true = [1, 1, 1]
y_pred = [0, 0, 0]

# Default behavior: returns 0.0 and emits UndefinedMetricWarning
f1_score(y_true, y_pred)

# Explicit nan: suppresses warning, returns nan
f1_score(y_true, y_pred, zero_division=np.nan)

# Goal after fix: new parameter with nan as default
# f1_score(y_true, y_pred)                              # => nan (no confusing 0.0)
# f1_score(y_true, y_pred, replaced_undefined_by=0.0)  # => 0.0
# f1_score(y_true, y_pred, zero_division=np.nan)        # => nan + DeprecationWarning
```

### Solution Approach

The plan is to:
1. Add `replaced_undefined_by=np.nan` as new parameter to all affected functions
2. Keep `zero_division` with a `FutureWarning` deprecation when used
3. Update `_check_zero_division` (or add `_check_replaced_undefined_by`) to validate the new param
4. Update `_prf_divide` and `_warn_prf` to use the new parameter name in warning messages
5. Update `@validate_params` constraints for each function
6. Update all docstrings
7. Add/update tests in `sklearn/metrics/tests/test_classification.py`

**Affected functions (~9 total):**
- `precision_recall_fscore_support`
- `precision_score`
- `recall_score`
- `f1_score`
- `fbeta_score`
- `jaccard_score`
- `classification_report`
- (possibly `class_likelihood_ratios` if applicable)

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
