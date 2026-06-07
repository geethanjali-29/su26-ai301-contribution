# su26-ai301-contribution

# Contribution 1: Add compatibility test for $sampleRate (second pass)

**Contribution Number:** 1
**Student:** Geethanjali Nagaboina
**Issue:** https://github.com/documentdb/functional-tests/issues/208
**Status:** Phase I Complete

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
No test file exists for `$sampleRate`. The operator is untested in this 
compatibility framework, meaning any regression or incompatibility with MongoDB 
would go undetected.

### Affected Components
- `tests/aggregate/` — this is where the new test file will be added, following 
  the existing pattern of other aggregation operator tests in the repo.

---

## Understanding the Issue (Sections below are for Phase II — leave as-is for now)

## Reproduction Process
### Environment Setup
[To be completed in Phase II]

### Steps to Reproduce
[To be completed in Phase II]

### Reproduction Evidence
[To be completed in Phase II]

---

## Solution Approach
[To be completed in Phase II]

---

## Testing Strategy
[To be completed in Phase II]

---

## Implementation Notes
[To be completed in Phase II]

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
