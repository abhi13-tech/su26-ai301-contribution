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

**Status:** 🔲 Not Started

### Understanding the Codebase
> _How did you explore the project? What files/modules are relevant? What did you learn about the architecture?_

### Reproduction Process
> _Steps you took to reproduce the bug or verify the missing feature. What did you observe?_

### Solution Approach
> _What is your plan to fix or implement this? What alternatives did you consider?_

---

## Phase III: Implementation

**Status:** 🔲 Not Started

### Testing Strategy
> _How will you verify your fix works? What tests exist? What tests will you write?_

### Implementation Notes
> _Document your implementation decisions, blockers you hit, and how you resolved them._

### Key Files Changed
> _List the files you modified and briefly explain each change._

---

## Phase IV: Pull Request

**Status:** 🔲 Not Started

| Field | Details |
|-------|---------|
| **PR Link** | _To be filled_ |
| **PR Title** | _To be filled_ |
| **Date Submitted** | _To be filled_ |
| **Current Status** | _Open / Under Review / Merged / Closed_ |

### PR Summary
> _Summarize what your pull request does in 2-3 sentences, as if explaining to the project maintainer._

### Maintainer Feedback Log

| Date | Reviewer | Feedback | Action Taken |
|------|----------|----------|--------------|
| _-_  | _-_      | _-_      | _-_          |

---

## Weekly Progress Log

| Week | Date | What I Did | Blockers | Next Steps |
|------|------|------------|----------|------------|
| 1    | 2026-06-04 | Set up contribution README, made first contribution to first-contributions repo, selected and claimed scikit-learn issue #33712 | None | Understand codebase, reproduce issue, plan solution (Phase II) |
| 2    |      |            |          |            |
| 3    |      |            |          |            |
| 4    |      |            |          |            |

---

## Resources & References

- [First Contributions Repo](https://github.com/abhi13-tech/first-contributions)
- [My PR to first-contributions](https://github.com/firstcontributions/first-contributions/pulls)
- _Additional links added as I progress..._

---

*Last updated: June 2026*
