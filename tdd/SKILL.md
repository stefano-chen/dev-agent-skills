---
name: tdd
description: "Test-driven development loop (red, green, refactor) for code changes. Use whenever the task is implementing a feature, fixing a bug, adding or changing behavior, or refactoring code that has or should have tests, even if the user never says TDD or mentions tests. Not for docs, config-only edits, pure renames, or when the user asks to skip tests."
argument-hint: "[feature, bug or refactor to work on]"
---

# Test-Driven Development

Work in small loops: write one test, watch it fail, write just enough code to pass, clean up. A test you saw fail before the code existed is evidence that the code works. A test written afterwards only shows that the test agrees with the code, bugs included. That difference is the whole reason for the order.

If the user gave text when invoking this skill, that text is the task.

## Size the work first

Match the ceremony to the change. Applying every section below to a one-line fix wastes the user's time; skipping the test list on a large feature loses cases.

- **Trivial** (typo, message text, comment, rename: no logic changes): no new test. Run the related tests and stop.
- **Small** (a one-line bug fix, a single function): one or two tests, the loop, run the related tests, report in a line or two. Skip the test list and the bug-hunting section, and baseline only the related tests. A one-line bug fix still gets its regression test.
- **Medium** (a feature inside one module): the full loop with a test list.
- **Large** (crosses modules, or the requirements are fuzzy): write the test list first. In an interactive session, show it to the user and wait for their go-ahead before coding. In an unattended run, or when the list comes from a plan the user already approved, put it in the report and continue. Build one thin end-to-end slice at a time. When an approved brief gives a build order, its steps are the slices, in its order. A step that adds behavior runs the loop. A step that only changes structure follows "Refactoring existing code". A step that pins existing behavior follows "Changing code that has no tests". Finish each step at green before starting the next.

## With other skills

This skill is about the order of work and the evidence. When other skills are active for the same task:

- An approved brief or spec decides what to build. Start the test list from its test plan and do not ask for approval of the list again. What it decided about the test setup and about commits replaces the defaults in this file. If it leaves a product or design decision open, ask instead of choosing; implementation details it does not mention are yours.
- A code-quality skill decides style and scope; a project-structure skill decides where test files go when the project has no convention.
- Give one final report: the shape under "Finishing", plus anything another skill asks you to report (trade-offs, things noticed but left out of scope).

## Pick the starting point

The loop in sections 1 to 3 assumes new behavior. Other kinds of work start somewhere else, so decide which this is before writing the first test:

- **New behavior:** the loop, sections 1 to 3.
- **Bug fix:** "Bug fixes: reproduce first".
- **Refactor** (structure changes, behavior doesn't): "Refactoring existing code". There is no red step.
- **Code with no tests:** "Changing code that has no tests" first, then whichever of the above applies.

All of them begin with Orient.

## 0. Orient (once per task)

1. **Find the test setup.** Look for the framework, config and existing tests (`package.json` scripts, `pyproject.toml`, `pytest.ini`, `go.mod`, `Cargo.toml`, `Makefile`, CI config). Copy the project's conventions for file location, naming, fixtures and assertion style, so the new tests look like they belong.
2. **If there is no test setup,** don't quietly pick a framework and add dependencies. If an approved brief or spec already decided this (which runner, or no setup at all), follow it. Otherwise prefer the runner built into the language (`unittest`, `go test`, `cargo test`, `node:test`). If a new dependency is needed, ask which one when the user is there to answer and there are real alternatives. Otherwise take the default for the project's stack and name it, with the reason, in the report. For trivial or small work in a project with no tests at all, do not introduce a test setup unasked: verify by running the code, say that no test was added, and offer to set one up.
3. **Find the fastest test command**, one that runs a single test or file (`pytest path::test_name -x`, `npx vitest run file -t name`, `go test ./pkg -run TestName`, `cargo test name`). It will run dozens of times.
4. **Get a baseline.** Run the existing suite once and note any tests that already fail, so they aren't confused with failures you cause. If the suite takes more than a couple of minutes, or the work is small, baseline only the packages you will touch and save the full run for the end.
5. **If tests can't run at all** (missing dependencies, broken environment), stop and tell the user. Writing tests that never execute is guessing, not TDD.
6. **Write a test list** (medium and large work). Turn the request into concrete behaviors, each one a single test:
   - the happy path
   - edge cases: empty, zero, one, many, max/min values, unicode, duplicates, ordering
   - invalid input and error paths
   - boundaries (`<` vs `<=`, first and last element)
   - state and interaction cases (called twice, concurrent calls, idempotency) where relevant

   Keep it somewhere that outlasts a long conversation (the task tracker if the environment has one, otherwise a notes file in a temp directory outside the source tree) and add to it whenever implementation reveals a new case.

If a requirement is ambiguous in a way that is cheap to change later, write the test for the interpretation you chose and say so. The test records the decision. Ask instead when the work follows an approved brief or spec and the point is a product or design decision it does not cover, or when the choice is a product decision that would be expensive to reverse.

## 1. RED: write one failing test

- Pick the **simplest** item left on the list.
- Write **one** test for **one** behavior. Name it after the behavior (`returns_empty_list_when_no_matches`), not the function.
- Assert on observable behavior (outputs, returned errors, state through the public API), not internals. Tests tied to internals break on every refactor and stop being trusted.
- Use specific expected values (`assert total == 42.50`), not "truthy" or "no exception".
- Test at the lowest level that shows the behavior through a public interface, because fast tests keep the loop short. Go up a level (integration, end to end) only when the behavior exists only once the parts are combined.
- **Run it and confirm it fails for the right reason:** the assertion, or "not implemented". A typo, a broken fixture or a failed import of something that should exist is the wrong reason; fix the test first.
- When the function, class or module under test does not exist yet, in any language, add the smallest stub first (returns a zero value or throws "not implemented") so the test file loads and the test fails on its assertion. A missing symbol is a build error in compiled languages and an import error in dynamic ones; either way it hides whether the assertion works, and in dynamic languages it also takes down every other test in the file.
- **If it passes right away, stop.** Either the behavior already exists or the test isn't testing what you think. Find out which by briefly breaking the code the test should depend on. If the test goes red, the behavior was already there: restore the code (see "Undoing"), keep the test if it adds coverage, and move to the next item. If it stays green, the test is wrong; fix it.

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

**Undoing.** "Go back to the last green state", "undo the last change" and "restore the code" all mean reversing your own edits from this step, by putting the previous text back. Do not use `git checkout`, `git restore`, `git reset` or `git stash` for this. Unless you have been committing at green points, they would also throw away the earlier green cycles, and in any case they can discard uncommitted work the user had. After restoring, re-run the tests to confirm you are back at green.

Putting the previous text back only works while you still have it exactly, which a long session or summarized context can take away. So in medium and large work where you are not committing, keep a fallback: at each green point, copy the files this task has changed or created into a temp directory outside the source tree, replacing the previous copy. Restoring is then copying those files back and deleting any file created since. Remove the copy when the work is finished.

**One test at a time.** Don't write all the tests up front and then all the code. Tests written in a batch are written against a design you imagined; each cycle teaches you something about the real one, and the later tests should benefit from it.

**Show the evidence.** Never report a test result you did not just see in real output. When you tell the user about a red or green step, include the command and the failing or passing line, not a paraphrase. There is no need to narrate every cycle; the final report and anything surprising are enough.

## Rules that keep the tests honest

- **Don't weaken a test to make it pass.** Loosening an assertion, deleting a case, adding a skip, swallowing an exception or pasting the actual output in as the expected value all turn the test into decoration. If you believe a test is wrong, say why and fix it as a separate, deliberate step.
- **Intended behavior changes are different.** When the task changes what the code should do, existing tests that assert the old behavior must change. Update those tests first, watch them fail, then change the code, and say which tests changed and why. When the task removes behavior, delete the tests for it in the same change and say which ones went.
- **Don't special-case test inputs** in production code (`if input == "test-value"`). The implementation has to be general.
- **Don't mock the unit under test.** Mock only real boundaries (network, clock, filesystem, randomness, third-party services), and prefer fakes or in-memory versions where they exist. A test that mocks the thing it tests only tests the mock.
- **Keep tests deterministic.** Inject or freeze time and random seeds. No sleeps, no dependence on test order.
- **Don't rerun a flaky test until it passes.** A test that fails only sometimes is pointing at something real (time, ordering, shared state, concurrency). Running it again once or twice to learn whether a failure is intermittent, and whether it also happens without your change, is diagnosis and is fine; treating the lucky pass as the result is not. Find the source, or report it if it isn't yours.
- **Leave unrelated tests alone.** Report tests that were already failing; don't fix, skip or delete them as a side effect.
- **Keep the loop small.** One behavior per cycle. If a cycle needs more than about 20 to 30 lines of production code, split it.

## Bug fixes: reproduce first

1. Write a test that **reproduces the bug** and fails, showing the wrong behavior. Write it at the lowest level that shows the bug; if it only appears when parts are combined, test at that level.
2. Check that it fails *because of the bug* and not for another reason.
3. If you can't reproduce it, stop before changing code and report what you tried. A fix for a bug you never saw fail is a guess: make one only if the user wants it, and label it as unverified. An approved bug report that says the bug was not reproduced, with a fix option the user picked, counts as the user wanting it. In that case first try to make a test fail at the level of the suspected defect; if it fails for the reason the report gives, that is the reproduction.
4. Fix the code and watch the test go green.
5. Ask **where else this bug could live.** Search for the same pattern and report what you find. Test and fix those places only when the request covers them or the user agrees; unrequested fixes elsewhere widen the change beyond what was asked.
6. Keep the regression test and name it after the bug.

## Hunting bugs (medium and large work)

After each feature on the list is green, spend a minute trying to break it:

- **Mutation check.** Break the code on purpose: flip a `<` to `<=`, drop a branch, return early. If no test fails, a test is missing; add it. Then restore the code (see "Undoing"), re-run the tests, and confirm the mutation is gone and only your intended change remains: check `git diff` for tracked files, and re-read the mutated lines in any file that is new, because untracked files do not appear in `git diff`.
- **Property tests.** For parsers, serializers, sorting, math and data transforms, add property-based tests if the project already has a library for them (Hypothesis, fast-check, proptest). Typical properties: round-trip (`decode(encode(x)) == x`), idempotence, "output is sorted", "length is preserved".
- **Error paths.** Every `raise`, `throw`, error return or `catch` you added should have a test that triggers it.
- **Read the diff as a reviewer.** For each changed line, ask which test would fail if it were wrong. Lines no test would catch are where bugs hide.

## Refactoring existing code

A refactor changes structure and keeps behavior, so there is no new failing test to write. The existing tests are the evidence, which only holds if they cover the code being moved.

1. Run the tests for the code you will touch and confirm they are green.
2. Check that they would notice a mistake: look at coverage if it is cheap, or break the code on purpose and see a test fail. Where nothing fails, add characterization tests first (next section).
3. Refactor in small steps, running the tests after each one, as in section 3.
4. Don't change a test's assertions during a refactor. If a test breaks because it was tied to internals (a renamed function, a moved import), make the mechanical fix and say so. If an expected value has to change, the behavior changed and this is no longer a refactor; treat it as an intended behavior change.

## Changing code that has no tests

Before changing untested code, write **characterization tests** that pin down what it does now, odd parts included. Get them green against the unchanged code, then change it under their protection. Pin the behavior you are about to change and what that change could break, not the whole module; for small work that can be a single test.

These tests pass on the first run by design, so the usual "watch it fail" evidence is missing. Get it another way: break the code briefly, confirm the test notices, restore it (see "Undoing") and confirm the test is green again.

If the code can't be tested as written (hidden dependencies, globals, hard-wired I/O), make the smallest mechanical change that opens it up, such as passing the dependency in as a parameter, and nothing more until tests exist.

If you find a real bug while doing this, pin the current behavior anyway, note the bug and report it. Fix it, separately and with the bug-fix flow above, only when it is part of the request, blocks it, or the user agrees.

## Finishing

1. Run the **full test suite**, plus the linter or type checker if the project has one. For small work with a slow suite, the related tests are enough; say that the full suite was not run.
2. Compare against the baseline. Any new failure is yours to fix.
3. If coverage is cheap to run, look at uncovered lines in the code you added.
4. Don't commit unless the user asked or an approved brief says to. When committing, make commits at green points, tests and code together, and give a refactor or a file move its own commit, apart from behavior changes.
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
