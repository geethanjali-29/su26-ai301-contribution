# su26-ai301-contribution
# Contribution 1: Add compatibility test for $sampleRate (second pass)
**Contribution Number:** 1
**Student:** Geethanjali Nagaboina
**Issue:** https://github.com/documentdb/functional-tests/issues/208
**Status:** Phase II Complete
---
## Why I Chose This Issue
I chose this issue because it is well-scoped and beginner-friendly, making it 
a great fit for my first open source contribution. The task asks me to write a 
compatibility test for the `$sampleRate` aggregation operator in DocumentDB — 
something I can accomplish entirely in Python using pytest, which I'm already 
familiar with. There's no need to modify engine code or work in an unfamiliar 
language, so I can focus on learning the contribution workflow itself.
I'm also genuinely interested in learning how database systems are tested at 
scale. This issue gives me a window into how DocumentDB validates compatibility 
with MongoDB's specification, which is a real-world engineering problem. By the 
end of this contribution, I expect to understand how aggregation operators work, 
how probabilistic behavior is tested reliably, and how to submit a production-
quality pull request to an active open source project.
---
## Understanding the Issue
### Problem Description
The `$sampleRate` aggregation pipeline operator in DocumentDB currently has no 
compatibility test coverage. Without a test, there is no automated way to verify 
that DocumentDB's implementation of `$sampleRate` behaves consistently with 
MongoDB's specification.
### Expected Behavior
A compatibility test should exist that:
- Inserts a set of documents into a collection
- Runs an aggregation pipeline using `{ $match: { $sampleRate: <rate> } }`
- Verifies that the output is a valid subset of the input documents
- Verifies that the operator accepts valid probability values (between 0.0 and 1.0)
- Verifies edge cases such as a rate of 0 (no documents) and 1.0 (all documents)
### Current Behavior
No test file existed for `$sampleRate`. The operator was untested in this 
compatibility framework, meaning any regression or incompatibility with MongoDB 
would go undetected.
### Affected Components
- `documentdb_tests/compatibility/tests/core/operator/expressions/misc/sampleRate/`
  — new directory and test file added here, following the pattern of other
  operator tests in the repo.
---
## Reproduction Process
### Environment Setup
- macOS 14, Python 3.11.7, pytest 7.4.0
- Cloned the fork and installed dependencies from repo root:
- Started local MongoDB on default port 27017
- Discovered that pytest must be run from inside `documentdb_tests/` (not the
  repo root) because `conftest.py` and `pytest.ini` live there — running from
  root causes `unrecognized arguments: --connection-string --engine-name` error

### Steps to Reproduce
1. Fork and clone the repository
2. From repo root: `pip install -e ".[dev]"`
3. Start MongoDB locally: `mongod --port 27017`
4. `cd documentdb_tests`
5. Run `find ~/functional-tests -name "*sampleRate*"` — no results, confirms test is missing
6. Run: pytest --connection-string mongodb://localhost:27017 --engine-name mongodb --collect-only
Result: 29,932 tests collected, zero for `$sampleRate`
7. This confirms the issue: no compatibility test exists for `$sampleRate`

### Reproduction Evidence
Branch: https://github.com/geethanjalinagaboina/functional-tests/tree/fix-issue-208

---

## Solution Approach

### Implementation Plan (UMPIRE)

- **Understand:** No smoke test exists for the `$sampleRate` aggregation operator.
  The operator takes a probability value (0.0–1.0) and randomly includes
  documents in pipeline output. Without a test, regressions go undetected.

- **Match:** Existing tests in
  `documentdb_tests/compatibility/tests/core/operator/expressions/misc/`
  follow a consistent pattern — insert documents, run an aggregation command
  via `execute_command`, assert results with `assertSuccess`. The `$rand` test
  in the same folder was used as the direct template.

- **Plan:**
  1. Create directory `sampleRate/` inside the `misc/` folder
  2. Create `test_smoke_sampleRate.py` following the existing pattern
  3. Insert 3 documents into the test collection
  4. Run `$match: {$sampleRate: 1.0}` — with rate 1.0, all documents must be returned
  5. Assert result matches expected documents using `assertSuccess`

- **Implement:** Branch `fix-issue-208`

- **Review:** Matches existing test style; uses `execute_command` and
  `assertSuccess` from the framework; marked with `pytest.mark.smoke`

- **Evaluate:** Running the test file in isolation gives 1 passed in 0.47s ✅

---

## Testing Strategy
Ran the new test file in isolation against a local MongoDB instance:
cd documentdb_tests
pytest --connection-string mongodb://localhost:27017 --engine-name mongodb 

compatibility/tests/core/operator/expressions/misc/sampleRate/test_smoke_sampleRate.py -v


Result: **1 passed in 0.47s**

The test verifies that with `$sampleRate: 1.0`, all inserted documents are
returned, confirming the operator is functional and compatible.

---

## Implementation Notes
- pytest must be invoked from `documentdb_tests/`, not the repo root
- The `result_analyzer/test_analyzer.py` file uses an unregistered `unit` marker
  which causes a collection error — use
  `--ignore=compatibility/result_analyzer/test_analyzer.py`
  if running the full suite
- `$sampleRate` is used inside a `$match` stage, not `$project` — it is a
  match expression that probabilistically filters documents

---

## Pull Request
[To be completed in Phase IV]

---

## Learnings & Reflections
[To be completed at end of program]

---

## Resources Used
- https://github.com/documentdb/functional-tests/issues/208
- https://www.mongodb.com/docs/manual/reference/operator/aggregation/sampleRate/
- https://github.com/documentdb/functional-tests/blob/main/README.md

