# Test Quality Review — <project or module>

Mode: Test Quality Review
Health Score: <0–100>

## Test Suite Map
<code block: unit/integration/e2e counts, ratio, coverage areas, Collection facts>

## Overall Assessment
1. Best behavior contracts (3–5 tests, and why they survive refactors):
2. Silent interference (never red, but steer implementation):
3. External behavior with no contract test if this module is refactored:
4. Tests to move to probes/ or delete:

## Before You Start: Build the Test Suite Map

Before filling the map, read these facts from the repo (do not guess):
- `testpaths` from `pyproject.toml` / `pytest.ini` / `setup.cfg`
- whether a `conftest.py` exists, and what fixtures it shares across files
- files skipped at collection time (`importorskip`, `skipif`, markers) — these are
  zero-collection tests, not coverage
- whether a `probes/` or similar scratch directory exists, and whether it is gitignored

Record these under the map as "Collection facts". They feed Step 5.

Then map the current test suite structure:

```
Unit tests:        X files, ~N tests
Integration tests: X files, ~N tests
E2E tests:         X files, ~N tests
Ratio:             Unit X%  :  Integration X%  :  E2E X%
Coverage areas:    [modules with tests] vs [modules without tests]
```

If you cannot access test files directly, ask the user **one question** — choose the
most relevant:
1. "Which module is hardest to test or has the least coverage?"
2. "When you make a change, how often do unrelated tests break?"
3. "Is there a part of the codebase your team avoids touching because it has no tests?"

After one answer, proceed. Do not ask more than one question.

---

## Analysis Process

Work through these six steps in order. Step 6 is the output step, not a scan.

### Step 1: Scan for Test Obscurity

*Scan this first — the most visible risk and the one that determines whether the suite
is maintainable at all.*

Look for:
- Read 5–10 test names at random: can each one communicate subject + scenario + expected
  outcome without opening the test body?
- Are there tests where a failure gives no clue which behavior broke (multiple assertions,
  no message strings)?
- Does any test depend on external state (files, database rows, env variables, shared mutable
  fixtures) that is invisible from within the test body?
- Is there a single massive setUp or beforeEach that every test inherits regardless of
  what it actually needs?

If all test names are clear and setups are minimal → no finding.

### Step 2a: Scan for Test Brittleness

*Brittle tests break on refactors that do not change observable behavior — they test
implementation, not contracts.*

Look for:
- Ask (or check git history): did any recent refactor cause test failures with no
  behavior change?
- Are there test methods where the name contains "and" or that assert on 3 or more
  unrelated behaviors (Eager Test)?
- Do assertions specify mock call order or exact parameter values that are irrelevant
  to the observable behavior?
- Are tests coupled to private methods or internal state directly?
- Agent-interference check: are there tests that rarely fail but make an automated
  or unfamiliar contributor avoid a reasonable refactor — e.g. asserting an internal
  call sequence, a private attribute, or an exact intermediate value that no caller
  observes? These are silent interference, not just brittleness: they never go red,
  but they steer implementation. Flag 🟡 with the refactor they would block.

**T2 Severity Guide (Brittleness):**
- 🔴 A recent behavior-preserving refactor broke tests, or tests assert private
  methods/internal state, or the suite would block a planned extraction.
- 🟡 Tests assert call order / exact intermediate values that no caller observes,
  or names bundle multiple behaviors ("and"), but no refactor is currently blocked.
- 🟢 Isolated brittle assertion with a clear local rewrite.

### Step 2b: Scan for Mock Abuse

*Mock Abuse produces tests that pass regardless of whether the real behavior is correct.
Scan this separately from brittleness — over-mocking is often the cause of brittleness,
but it is a distinct problem worth its own finding.*

**Sample 3–5 tests once for both steps 2a and 2b together** — read each test body and
check brittleness signals and mock-setup ratio in the same pass, then write separate
findings if both problems are present.

Look for:
- Is mock setup code longer than the assertion logic in the sampled tests?
- Are the primary assertions `expect(mock).toHaveBeenCalledWith(...)` rather than
  assertions on outputs, state, or observable events?
- Are there methods in production classes that are only called from test files
  (test-induced design damage)?
- Does any single test create more than 3 mock objects?

**Positive principle:** mocks belong only on uncontrolled external systems (network,
clock, filesystem, third-party APIs) and on narrow interfaces the codebase owns.
Internal collaboration uses fakes or real lightweight implementations. A test that
mocks an internal collaborator to assert its calls is testing the mock contract, not
the behavior contract.

**T4 Severity Guide (Mock Abuse):**
- 🔴 Production methods exist only for test access, or primary assertions are
  `toHaveBeenCalled*` instead of outputs/state/events.
- 🟡 Mock setup-to-assertion ratio exceeds 3:1, or a single test creates >3 mocks,
  but assertions still land on observable behavior.
- 🟢 A handful of mocks on true external boundaries with clear justification.

### Step 3: Scan for Test Duplication

Look for:
- Is the same setup block (same variables initialized the same way) repeated across
  5 or more test files without a shared helper?
- Are there multiple tests that pass identical inputs and assert identical outputs
  with no differentiation (Lazy Test)?
- Is the same business scenario covered at unit, integration, and E2E level with no
  difference in what each layer is testing?

Parameterization quality:
- Good: one test function, multiple input sets, identical assertion logic — same
  behavior contract, different data.
- Bad: different inputs take different behavior branches inside the same test —
  that is several contracts wearing one name. Split it, do not parameterize it.

**T3 Severity Guide (Duplication):** instance counts are the tiebreaker —
🔴 10+ instances, 🟡 3–9, 🟢 1–2.

### Step 4: Scan for Coverage Illusion and Architecture Mismatch

Look for Coverage Illusion:
- Pick the most recently modified core module. Are its error-handling branches and
  null/boundary inputs covered by tests?
- Are there legacy areas (old functions, no test files nearby) that are actively
  being changed?
- Do the tests assert on side effects (DB writes, events emitted, state transitions)
  or only on return values?

**Characterization Test check:** If legacy code is being modified without existing tests,
the team needs Characterization Tests before making the change — not after. Look for
this pattern and flag it when absent.

A Characterization Test locks in current behavior (right or wrong) so future changes
do not silently regress it. Template:
```
test("characterize: [module].[method] given [input], returns [current output]") {
  // Call the code under test with realistic inputs
  // Assert on whatever it currently returns — even if you suspect the output is wrong
  // Add a comment: "This captures current behavior, not necessarily correct behavior"
}
```
Source: Feathers — Working Effectively with Legacy Code, Ch. 13: Characterization Tests

Look for Architecture Mismatch:
- Compare the suite map from the start: is the ratio close to 70% unit / 20% integration / 10% E2E?
- Are high-risk modules tested at higher density than trivial utilities?

**Test suite performance:** A slow test suite is a first-class maintainability risk — it
breaks the fast-feedback loop and causes developers to skip running tests locally.
- If the full suite runtime is > 10 minutes → 🟡 Warning (runtime alone does not reach Critical)
- Runtime > 30 minutes, or unknown because nobody runs the suite, is still 🟡 Warning —
  but say so explicitly in the Symptom. It reaches 🔴 Critical only when combined with a
  Critical condition: legacy code under modification with no seams and no
  characterization tests, or a fully inverted pyramid
- If tests that could be unit tests are integration tests, that is a Performance Mismatch:
  each misclassified test adds seconds of avoidable wait time

**T6 Severity Guide (Performance):**
- 🔴 Slow suite + legacy code being modified without seams/characterization tests,
  or a fully inverted test pyramid.
- 🟡 Full suite > 10 minutes, or > 30 minutes, or unknown because nobody runs it.
- 🟢 No avoidable integration-level overhead in tests that could be unit tests.

Source: Meszaros — xUnit Test Patterns, Slow Tests (p. 253)

### Step 5: Scan for Lifecycle Debt

*Tests that occupy maintenance cost without providing protection. A test that does
not run, or only runs on one machine, is worse than no test: it signals coverage
that does not exist.*

Look for:
- Zombie tests: source files present but zero tests collected (importorskip on a
  missing optional dependency, skipif on a personal path). Confirm against the
  Collection facts gathered in "Before You Start".
- Environment-coupled probes: hard-coded personal paths, developer-machine
  directories, network assumptions. These fail on another machine or are silently
  skipped — either way they do not protect.
- Golden files without an update path: baseline/snapshot files with no explicit
  update command. Intentional changes make many tests red at once and invite
  "edit the baseline to go green". Require an `UPDATE_*` entry point and a reviewed
  baseline diff.
- Scratch probes inside the permanent suite: temporary capture tests, one-off
  environment checks, anything with no named behavior contract. These belong in a
  gitignored probes directory, not in `tests/`.

**T7 Severity Guide (Lifecycle Debt):**
- 🔴 Zombie tests or environment-coupled tests sitting in the permanent suite.
- 🟡 Golden files without an update path, or scratch probes leaking into `tests/`.
- 🟢 Lifecycle risk identified and already isolated in a probes directory.

### Step 6: Apply Iron Law, Output Report

For each finding, apply the Iron Law format from the Report Contract:
Symptom → Source → Consequence → Remedy, with a severity label.

Use the Report Template from the Report Contract. Mode: Test Quality Review.

Immediately before the `## Findings` heading, output in this order:
1. The Test Suite Map as a code block, labeled "Test Suite Map".
2. A short **Overall Assessment** answering:
   - Which 3–5 tests are the best behavior contracts — and why they survive refactors?
   - Which tests are silent interference — never red, but steer implementation?
   - Which external behavior has no contract test if this module is refactored?
   - Which current tests should move to `probes/` or be deleted? Give the list.

Do not turn the report into a pure defect list; the assessment is what makes it
actionable. Compute the Health Score per the Report Contract and show finding counts
next to it.

**唯一需要你决定的是 T4 这个编号。** 原来 `test-decay-risks.md` 不存在，所以 T2/T3/T6 这三个编号是从旧引用里保留下来的，T4（Mock Abuse）和 T7（Lifecycle Debt）是我新起的。如果你希望编号连续无跳号，把 T4 改成 T5、T7 改成 T6 即可，guide 里各出现一次，改动很小。
