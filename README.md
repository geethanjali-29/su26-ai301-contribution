# su26-ai301-contribution
# Contribution 1: Add compatibility test for $sampleRate (second pass)

**Contribution Number:** 1
**Student:** Geethanjali Nagaboina
**Issue:** https://github.com/documentdb/functional-tests/issues/208
**Status:** Phase II Complete

---

## Why I Chose This Issue

I chose this issue because it is well-scoped and beginner-friendly, making it a great fit for my first open source contribution. The task asks me to write a compatibility test for the `$sampleRate` aggregation operator in DocumentDB — something I can accomplish entirely in Python using pytest, which I'm already familiar with. There's no need to modify engine code or work in an unfamiliar language, so I can focus on learning the contribution workflow itself.

I'm also genuinely interested in learning how database systems are tested at scale. This issue gives me a window into how DocumentDB validates compatibility with MongoDB's specification, which is a real-world engineering problem. By the end of this contribution, I expect to understand how aggregation operators work, how probabilistic behavior is tested reliably, and how to submit a production-quality pull request to an active open source project.

---

## Understanding the Issue

### Problem Description

The `$sampleRate` aggregation pipeline operator in DocumentDB currently has no compatibility test coverage. Without a test, there is no automated way to verify that DocumentDB's implementation of `$sampleRate` behaves consistently with MongoDB's specification. This means any regression or incompatibility would go undetected in CI.

### Expected Behavior

A compatibility test should exist that:
- Inserts a known set of documents into a collection
- Runs an aggregation pipeline using `{ $match: { $sampleRate: <rate> } }`
- Verifies that the output is a valid subset of the input documents
- Verifies that the operator accepts valid probability values (between 0.0 and 1.0)
- Verifies edge cases: a rate of `0.0` (no documents returned) and `1.0` (all documents returned)

The test should live at:
`documentdb_tests/compatibility/tests/core/operator/expressions/misc/sampleRate/test_smoke_sampleRate.py`

### Current Behavior

No test file existed for `$sampleRate`. Running `find ~/functional-tests -name "*sampleRate*"` returns zero results. Collecting the full test suite (29,932 tests) confirms zero coverage for this operator, meaning any regression or incompatibility with MongoDB's specification would go completely undetected.

### Affected Components

- **Missing file:** `documentdb_tests/compatibility/tests/core/operator/expressions/misc/sampleRate/test_smoke_sampleRate.py` — new directory and test file added here
- **Reference pattern:** `documentdb_tests/compatibility/tests/core/operator/expressions/misc/rand/test_smoke_rand.py` — used as the direct structural template
- **Framework utilities used:** `execute_command` and `assertSuccess` from `documentdb_tests/compatibility/` test framework
- **Config files consulted:** `documentdb_tests/conftest.py` and `documentdb_tests/pytest.ini` — required to understand how custom CLI flags (`--connection-string`, `--engine-name`) are registered

---

## Reproduction Process

### Environment Setup

**Setup approach:** I followed the repository's `README.md` instructions, then inspected `pytest.ini` and `conftest.py` to understand test runner configuration before writing any code.

**Challenges encountered and how I resolved them:**

1. **`unrecognized arguments: --connection-string --engine-name` error**
   - *Problem:* Running `pytest` from the repo root failed because `conftest.py` and `pytest.ini` (which register the custom CLI flags) live inside `documentdb_tests/`, not at the root.
   - *Resolution:* Always `cd documentdb_tests` before invoking pytest. This is not documented in the main README and cost significant debugging time.

2. **Unregistered `unit` marker causing collection error**
   - *Problem:* Running the full test suite triggered a collection-time error from `compatibility/result_analyzer/test_analyzer.py` due to an unregistered pytest marker.
   - *Resolution:* Pass `--ignore=compatibility/result_analyzer/test_analyzer.py` when running the full suite. For isolated file testing, this flag is not needed.

3. **Locating the correct test pattern**
   - *Problem:* The repo contains many test styles; it was unclear which pattern to follow for a new operator.
   - *Resolution:* Searched for neighboring operator tests under `misc/` and identified `test_smoke_rand.py` as the closest structural match since `$rand` is also a probabilistic expression.

**Final working environment:** macOS 14, Python 3.11.7, pytest 7.4.0, local MongoDB on port 27017.

### Steps to Reproduce (Missing Test Coverage)

1. Fork and clone the repository: `git clone https://github.com/geethanjalinagaboina/functional-tests.git`
2. From repo root, install dependencies: `pip install -e ".[dev]"`
3. Start a local MongoDB instance: `mongod --port 27017`
4. Change into the test directory: `cd documentdb_tests`
5. Confirm no test exists for `$sampleRate`:
   ```
   find ~/functional-tests -name "*sampleRate*"
   ```
   **Expected result:** One or more test files returned
   **Actual result:** No output — zero files found
6. Collect the full suite to confirm zero coverage:
   ```
   pytest --connection-string mongodb://localhost:27017 --engine-name mongodb --collect-only
   ```
   **Expected result:** At least one test for `$sampleRate` listed
   **Actual result:** 29,932 tests collected, none for `$sampleRate`

### Reproduction Evidence

- **Branch:** https://github.com/geethanjalinagaboina/functional-tests/tree/fix-issue-208
- **Issue confirmed:** Running `--collect-only` shows 29,932 tests with zero results when filtering for `sampleRate`, confirming the gap in coverage

---

## Solution Approach

### Implementation Plan (UMPIRE)

**Understand:**
The root cause is a missing test file — the `$sampleRate` operator exists in DocumentDB's engine but was never added to the compatibility test framework. Unlike a code bug, there is no broken function to fix; the gap is purely in test coverage. The operator takes a float probability value between `0.0` and `1.0` and probabilistically includes documents in pipeline output. Deterministic testing requires using the boundary values (`0.0` and `1.0`) where behavior is guaranteed.

**Match:**
Existing tests in `documentdb_tests/compatibility/tests/core/operator/expressions/misc/` follow a consistent pattern: insert documents → run aggregation via `execute_command` → assert with `assertSuccess`. The `$rand` test at `misc/rand/test_smoke_rand.py` was used as the direct structural template, as it is also a probabilistic expression and lives in the same folder.

**Plan:**
1. Create directory: `documentdb_tests/compatibility/tests/core/operator/expressions/misc/sampleRate/`
2. Create file: `test_smoke_sampleRate.py` modeled on `test_smoke_rand.py`
3. Insert 3 known documents into the test collection using `execute_command`
4. Run `{ $match: { $sampleRate: 1.0 } }` — at rate 1.0, all documents must be returned (deterministic)
5. Assert the result matches all 3 inserted documents using `assertSuccess`
6. Mark test with `@pytest.mark.smoke` consistent with neighboring tests

**Implement:**
Branch:  https://github.com/geethanjali-29/functional-tests/tree/fix-issue-208
**Review:**
- Matches existing test structure and naming conventions in `misc/`
- Uses only framework-provided utilities (`execute_command`, `assertSuccess`) — no new dependencies
- Marked `@pytest.mark.smoke` per project convention
- Does not modify any engine or framework code — purely additive

**Evaluate:**
Run the new test file in isolation:
```
cd documentdb_tests
pytest --connection-string mongodb://localhost:27017 --engine-name mongodb \
  compatibility/tests/core/operator/expressions/misc/sampleRate/test_smoke_sampleRate.py -v
```
**Result: 1 passed in 0.47s ✅**

---

## Testing Strategy

### What Was Tested

- **Rate = 1.0 (all documents):** With `$sampleRate: 1.0`, all 3 inserted documents must be returned. This is deterministic and confirms basic operator functionality.
- **Operator placement:** Confirmed `$sampleRate` is used inside a `$match` stage (not `$project`) — it is a match expression, not a projection expression.
- **Framework integration:** `execute_command` and `assertSuccess` correctly handle the aggregation result format.

### Manual Testing

Ran the full test file in isolation against a local MongoDB 7.x instance. Output:

```
compatibility/tests/core/operator/expressions/misc/sampleRate/test_smoke_sampleRate.py::test_smoke_sampleRate PASSED
1 passed in 0.47s
```

---

## Implementation Notes

- pytest must always be invoked from `documentdb_tests/`, not the repo root — `conftest.py` and `pytest.ini` are not at root
- When running the full suite, add `--ignore=compatibility/result_analyzer/test_analyzer.py` to avoid an unregistered `unit` marker error
- `$sampleRate` is a **match expression**, not a projection operator — it must appear inside `{ $match: { $sampleRate: <rate> } }`
- Testing probabilistic operators deterministically requires using the guaranteed boundary values: `0.0` (no documents) and `1.0` (all documents)

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
