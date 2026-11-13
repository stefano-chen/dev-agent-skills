---
name: tdd
description: "Test-driven development loop (red, green, refactor) for code changes. Use whenever the task is implementing a feature, fixing a bug, adding or changing behavior, or refactoring code that has or should have tests, even if the user never says TDD or mentions tests. Not for docs, config-only edits, pure renames, or when the user asks to skip tests."
argument-hint: "[feature, bug or refactor to work on]"
---

# Test-Driven Development

Work in small loops: write one test, watch it fail, write just enough code to pass, clean up. A test you saw fail before the code existed is evidence that the code works. A test written afterwards only shows that the test agrees with the code, bugs included. That difference is the whole reason for the order.

If text follows `/tdd`, that text is the task.

## Size the work first

Match the ceremony to the change. Applying every section below to a one-line fix wastes the user's time; skipping the test list on a large feature loses cases.

- **Small** (one-line fix, a single function): one or two tests, the loop, run the related tests, report. Skip the test list and the bug-hunting section.
- **Medium** (a feature inside one module): the full loop with a test list.
- **Large** (crosses modules, or the requirements are fuzzy): write the test list first and show it to the user before coding if they are around. Build one thin end-to-end slice at a time.

## 0. Orient (once per task)

1. **Find the test setup.** Look for the framework, config and existing tests (`package.json` scripts, `pyproject.toml`, `pytest.ini`, `go.mod`, `Cargo.toml`, `Makefile`, CI config). Copy the project's conventions for file location, naming, fixtures and assertion style, so the new tests look like they belong.
2. **If there is no test setup,** don't quietly pick a framework and add dependencies. Prefer the runner built into the language (`unittest`, `go test`, `cargo test`, `node:test`). If a new dependency is needed, name it in one line and why, then carry on unless the user objects.
3. **Find the fastest test command**, one that runs a single test or file (`pytest path::test_name -x`, `npx vitest run file -t name`, `go test ./pkg -run TestName`, `cargo test name`). It will run dozens of times.
4. **Get a baseline.** Run the existing suite once and note any tests that already fail, so they aren't confused with failures you cause. If the suite takes more than a couple of minutes, baseline only the packages you will touch and save the full run for the end.
5. **If tests can't run at all** (missing dependencies, broken environment), stop and tell the user. Writing tests that never execute is guessing, not TDD.
6. **Write a test list** (medium and large work). Turn the request into concrete behaviors, each one a single test:
   - the happy path
   - edge cases: empty, zero, one, many, max/min values, unicode, duplicates, ordering
   - invalid input and error paths
   - boundaries (`<` vs `<=`, first and last element)
   - state and interaction cases (called twice, concurrent calls, idempotency) where relevant

   Keep it in the task list and add to it whenever implementation reveals a new case.

If a requirement is ambiguous, write the test for the interpretation you chose and say so. The test records the decision.

## 1. RED: write one failing test

- Pick the **simplest** item left on the list.
- Write **one** test for **one** behavior. Name it after the behavior (`returns_empty_list_when_no_matches`), not the function.
- Assert on observable behavior (outputs, returned errors, state through the public API), not internals. Tests tied to internals break on every refactor and stop being trusted.
- Use specific expected values (`assert total == 42.50`), not "truthy" or "no exception".
- **Run it and confirm it fails for the right reason:** the assertion, or "not implemented". A typo, import error or broken fixture is the wrong reason; fix the test first.
- In compiled or typed languages a missing symbol is a build error, which hides whether the assertion works. Add the smallest stub (returns a zero value or throws "not implemented") so the test compiles and fails on its assertion.
- **If it passes right away, stop.** Either the behavior already exists or the test isn't testing what you think. Find out which before moving on.

## 2. GREEN: make it pass with minimal code

- Write the **simplest code** that passes this test. Don't write code for tests that don't exist yet; untested code is exactly what this process is meant to prevent.
- A hard-coded return is acceptable as a stepping stone only when the next test will force the real implementation. It is never the finished state.
- Run the single test, then the **related tests** (the file or module) to catch regressions while the cause is still obvious.
- If GREEN takes more than a couple of attempts, the step is too big. Go back to the last green state and write a smaller test.

## 3. REFACTOR: clean up while green

- Look for duplication, poor names and tangled conditionals, in the tests as well. Concluding that nothing needs changing is fine; skipping the look is not.
- Run tests after **each** refactoring step. If anything goes red, undo the last change instead of debugging forward.
- Don't change behavior and structure in the same step, or a red test can't tell you which one broke it.

Then return to RED with the next item.

**One test at a time.** Don't write all the tests up front and then all the code. Tests written in a batch are written against a design you imagined; each cycle teaches you something about the real one, and the later tests should benefit from it.

**Show the evidence.** When you report a red or green step, include the command and the failing or passing line from the real output. Never report a test result you did not just see.

## Rules that keep the tests honest

- **Don't weaken a test to make it pass.** Loosening an assertion, deleting a case, adding a skip, swallowing an exception or pasting the actual output in as the expected value all turn the test into decoration. If you believe a test is wrong, say why and fix it as a separate, deliberate step.
- **Intended behavior changes are different.** When the task changes what the code should do, existing tests that assert the old behavior must change. Update those tests first, watch them fail, then change the code, and say which tests changed and why.
- **Don't special-case test inputs** in production code (`if input == "test-value"`). The implementation has to be general.
- **Don't mock the unit under test.** Mock only real boundaries (network, clock, filesystem, randomness, third-party services), and prefer fakes or in-memory versions where they exist. A test that mocks the thing it tests only tests the mock.
- **Keep tests deterministic.** Inject or freeze time and random seeds. No sleeps, no dependence on test order.
- **Leave unrelated tests alone.** Report tests that were already failing; don't fix, skip or delete them as a side effect.
- **Keep the loop small.** One behavior per cycle. If a cycle needs more than about 20 to 30 lines of production code, split it.

## Bug fixes: reproduce first

1. Write a test that **reproduces the bug** and fails, showing the wrong behavior. Write it at the lowest level that shows the bug; if it only appears when parts are combined, test at that level.
2. Check that it fails *because of the bug* and not for another reason.
3. If you can't reproduce it, say so before changing code. A fix for a bug you never saw fail is a guess, and should be labeled as one.
4. Fix the code and watch the test go green.
5. Ask **where else this bug could live.** Search for the same pattern and add tests for those places.
6. Keep the regression test and name it after the bug.

## Hunting bugs (medium and large work)

After each feature on the list is green, spend a minute trying to break it:

- **Mutation check.** Break the code on purpose: flip a `<` to `<=`, drop a branch, return early. If no test fails, a test is missing; add it. Then restore the code and confirm with `git diff` that the mutation is gone.
- **Property tests.** For parsers, serializers, sorting, math and data transforms, add property-based tests if the project already has a library for them (Hypothesis, fast-check, proptest). Typical properties: round-trip (`decode(encode(x)) == x`), idempotence, "output is sorted", "length is preserved".
- **Error paths.** Every `raise`, `throw`, error return or `catch` you added should have a test that triggers it.
- **Read the diff as a reviewer.** For each changed line, ask which test would fail if it were wrong. Lines no test would catch are where bugs hide.

## Changing code that has no tests

Before changing untested code, write **characterization tests** that pin down what it does now, odd parts included. Get them green against the unchanged code, then change it under their protection.

If the code can't be tested as written (hidden dependencies, globals, hard-wired I/O), make the smallest mechanical change that opens it up, such as passing the dependency in as a parameter, and nothing more until tests exist.

If you find a real bug while doing this, note it and fix it separately with the bug-fix flow above.

## Finishing

1. Run the **full test suite**, plus the linter or type checker if the project has one.
2. Compare against the baseline. Any new failure is yours to fix.
3. If coverage is cheap to run, look at uncovered lines in the code you added.
4. Don't commit unless the user asked. If they want commits, make them at green points, tests and code together.
5. Report in this shape, keeping it short:

```
Tests added: <n>, all passing. Full suite: <result vs. baseline>.
Behaviors covered: <the test list, briefly>
Bugs caught along the way: <what the tests found, or "none">
Not handled: <edge cases left out and why, or "nothing">
Run with: <exact command>
```

## When to relax the loop

Strict red-green is the default. For exploratory spikes, throwaway prototypes, or pure config and wiring with no logic, code may come first. Say that you are doing so, and add tests before calling the work done if the code will stay. If the user says they don't want tests, respect that and say what is left unverified.
