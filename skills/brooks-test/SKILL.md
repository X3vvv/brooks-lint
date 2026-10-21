---
name: brooks-test
description: >
  Test quality review drawing on twelve classic engineering books — with primary focus
  on xUnit Test Patterns, The Art of Unit Testing, How Google Tests Software, and
  Working Effectively with Legacy Code — that diagnoses structural problems in an
  existing test suite: brittleness, mock abuse, coverage illusions, slow execution,
  poor readability, and test-lifecycle debt (zombie tests, environment-coupled probes,
  golden files without an update path).
  Triggers when: user asks about test quality, shares test files for review, or
  expresses frustration: "tests keep breaking whenever I change anything", "our tests
  take forever", "tests pass but bugs still reach production", or "we have too many mocks".
  Do NOT trigger for: writing new tests from scratch (use the regular test-writing
  workflow) or testing framework/syntax questions — this skill reviews an existing
  suite for structural quality problems, not individual test authoring.
metadata:
  opencode/slash: "true"
---

# Brooks-Lint — Test Quality Review

## Setup

Read in order:

1. `test-guide.md` (this directory) — the complete review framework. It is self-contained:
   it defines Auto Scope Detection, the Root Criterion, the Iron Law report format,
   Health Score, source references, severity guides (T2/T3/T4/T6/T7), the Mock positive
   principle, and the Lifecycle Debt scan.

No other files are required.

## Process

1. Determine scope using the guide's "Report Contract → Auto Scope Detection" section.
2. Build the test suite map and Collection facts (guide's "Before You Start" section).
3. Scan for each test decay risk in the order specified (guide's Steps 1–5), applying
   the Root Criterion before each scan.
4. Apply the Iron Law and output using the Report Template (guide's Step 6).

**Mode line in report:** `Test Quality Review`
