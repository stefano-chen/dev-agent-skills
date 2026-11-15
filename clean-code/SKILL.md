---
name: clean-code
description: "Standards for writing, changing, refactoring and reviewing code so it stays readable, minimal and consistent with the codebase it lives in. Use for any task that produces or changes code (implementing a feature, fixing a bug, refactoring, cleaning up, writing tests, reviewing a diff or pull request), even when the user never mentions code quality, and whenever the user asks for SOLID, DRY, KISS, YAGNI or \"best practices\". Not needed for questions that only ask to explain code, or for throwaway one-off commands."
---

# Clean Code

Clean code is code the next reader can understand, change and trust quickly. You already know the textbook principles (SOLID, DRY, KISS, YAGNI). This skill is about the judgment calls between them, and about the habits that most often make an otherwise correct change worse.

## What wins when goals conflict

1. **What the user and the project say.** Explicit instructions, the project's instruction files for coding agents (`AGENTS.md`, `CLAUDE.md` and the like), `CONTRIBUTING.md` and linter/formatter configs override the choices below.
2. **Correctness.** Clean code that does the wrong thing is worthless.
3. **Consistency with the surrounding code.** A "better" pattern used in one file makes the codebase harder to read, not easier. Match what is there (naming, layout, error mechanism, test style), even where this skill would choose differently, and mention the disagreement instead of acting on it.
4. **Simplicity.** The least code and the fewest concepts that fully solve the problem.
5. **The habits and defaults in this file.**

Two things sit outside this ranking, because neither an instruction nor a convention makes them acceptable:

- **Defects are not conventions.** Consistency covers style, not flaws. If the neighbors concatenate SQL, swallow errors, log secrets or are buried in noise comments and needless guards, do not copy that into new code. Write the new code correctly in the project's style, leave the existing code alone, and mention the pattern in your report.
- **Honesty about the result.** An instruction such as "just make CI green" is not a request to weaken a test, hide a failure or claim something works unverified. If the user explicitly asks for a shortcut (skip this test, hard-code that value), do it and say plainly what it leaves broken or unverified.

## With other skills

This skill is about what the code looks like. When other skills are active for the same task, each keeps its own subject:

- An approved brief or spec (for example from a design interview) decides scope and design decisions.
- A test-driven development skill decides the process for tests, bug fixes and refactors. The versions of those flows in this file are the short fallback for when it is not loaded.
- A project-structure skill decides where files go.
- Give one final report, not one per skill. If another skill prescribes a report format, use it and add whatever from "Before finishing" it lacks.

## Scale the effort to the change

Applying every practice to every change is its own kind of mess: a typo fix that arrives with a refactor and three new test files is harder to review than the typo was to live with.

| Change | What it needs |
| --- | --- |
| Trivial change that alters no logic (typo, message text, comment, rename) | Match the local style, run the relevant test. Nothing else, and no new tests. |
| Bug fix, of any size including one line | A regression test that fails without the fix, then the smallest fix that addresses the cause rather than the symptom. If the project has no tests at all, do not build a test setup for a small fix: verify it by running the code and say that no regression test exists. |
| New feature | Read neighbors first, reuse existing helpers, test the new behavior, update docs the change makes wrong. |
| Refactor | Behavior stays identical; see "Refactoring" below. |
| New module or project | The defaults below apply in full wherever there is no existing convention. A new module inside an existing project still follows that project's conventions. |

## Before writing

- **Read the neighbors.** Look at the files next to the one you are changing: naming, error handling, test style, folder layout. Five minutes here prevents a change that works but looks foreign.
- **Search before you create.** A helper, type or constant for this job often already exists. Duplicated logic drifts apart and becomes two bugs.
- **Fix the scope.** Decide what the request covers. Anything else you notice (a nearby bug, an ugly function) goes in your final report as a suggestion, not into the diff. Unrequested edits make changes hard to review and easy to break.

## Habits to catch in your own work

These are the most common ways a change ends up worse than it needed to be. Check for each before finishing.

- **Over-engineering.** Interfaces with one implementation (other than at a real boundary such as the database or network, see "Dependency inversion against YAGNI"), config options nobody asked for, factories and wrappers around a single call, generic "frameworks" for one use case. Write for today's requirement; abstractions are cheap to add later and expensive to remove.
- **Scope creep.** Renaming, reformatting or "improving" code the task did not touch. Reformatting untouched lines also buries the real change in the diff.
- **Comment noise.** Comments that restate the code, narrate the edit ("changed to use the new helper", "fix for the bug above"), or mark sections of an obvious function. A comment earns its place by saying *why*: a business rule, a trade-off, a non-obvious constraint.
- **Defensive clutter.** `try/catch` around code that cannot fail, null checks for values the type already guarantees, fallbacks that silently hide a broken state. Validate at the boundary where untrusted data enters, then trust it. Extra guards inside add branches nobody can test and hide real bugs.
- **Compatibility baggage.** Keeping the old function as a wrapper, re-exporting old names, leaving a "legacy" path, when nothing asked for backward compatibility. Change the call sites and delete the old code. This holds only when every caller is in code you can search. A published library API, HTTP endpoint, CLI flag, database schema or stored data format has callers you cannot see: keep it compatible, or ask before breaking it.
- **Leftovers.** Debug prints, commented-out code, unused imports and variables, scratch files, `TODO`s with no content. Version control remembers deleted code; the file should not.
- **Tests bent to pass.** Weakening an assertion, skipping a test, or special-casing test input in production code. If a test fails, either the code is wrong or the test's expectation is out of date; work out which and say so.
- **Unverified claims.** Saying "tests pass" or "this works" without having run anything. Run it, or state plainly what you could not run.

## When the principles pull against each other

SOLID, DRY, KISS and YAGNI each fix a real problem, and each causes one when applied on reflex. KISS and YAGNI are the starting point. DRY and SOLID are remedies for pain that has already shown up in the code, not structure to install in advance.

- **DRY against KISS.** A shared helper that needs a flag per caller is the wrong abstraction and costs more than the duplication did. Inline it back and let the callers differ.
- **Open/closed against YAGNI.** Do not build extension points for variants nobody has asked for. The evidence to wait for is editing the same `switch` or `if` chain a second or third time to add a case. At that point turn the variants into data or pluggable implementations, so the next one is an addition instead of an edit to working code.
- **Dependency inversion against YAGNI.** Invert the dependencies that cross a boundary (database, network, clock, filesystem), because that is what makes the logic testable. An interface in front of an in-process class with one implementation is only indirection.
- **Single responsibility against fragmentation.** "One reason to change" is about who would ask for the change (pricing rules versus invoice layout), not about size. Splitting cohesive code into tiny classes scatters one idea across many files.

## Defaults

Use these where the codebase has no convention of its own. The numbers are signals to stop and look, not limits to enforce: a clear 50-line function is better than five fragments the reader has to chase.

### Naming

- Names say what a thing is for: `elapsedDays` over `d`, `fetchActiveUsers()` over `getData()`.
- Functions are verbs, values are nouns, booleans read as questions (`isValid`, `hasAccess`).
- One word per concept across the codebase: pick `fetch` or `get` or `retrieve`, not all three.
- Name length follows scope: `i` is fine in a three-line loop, not for an exported value.
- Give magic numbers and strings a name (`MAX_RETRY_ATTEMPTS = 3`) when the meaning is not obvious from context.

### Functions

- One job each. If describing it needs "and", consider splitting.
- Guard clauses and early returns instead of deep nesting; more than about three levels of indentation is a signal.
- One level of abstraction per function: orchestration and byte-level parsing do not belong in the same body.
- More than about three parameters is a signal to group them into an options object or struct.
- A boolean parameter that switches behavior (`render(true)`) usually reads better as two named functions.
- Keep side effects (I/O, network, mutation) at the edges and the logic in between pure, because pure code is trivial to test.

### Structure

- Each module has one reason to change; keep things that change together close together.
- Remove duplicated *knowledge*, not code that merely looks alike. Two similar blocks that change for different reasons should stay separate; wait for the third real repetition before abstracting.
- Prefer composition to inheritance, and pass external dependencies (database, HTTP client, clock, randomness) in rather than reaching for them, so the code can be tested.
- A subclass or interface implementation must work everywhere the base type is accepted: no overrides that throw "not supported", reject inputs the base accepts, or return less than it promises. If it cannot honor the contract, it is a different type, not a subtype.
- Keep interfaces narrow and shaped by what each caller uses. A function that reads one field takes that field, not the whole object or a context bag, and an implementer should never have to stub methods it has no use for.
- Export only what callers need.

### Types and data

- Use the type system the project already uses, at the strictness it already uses. Avoid `any` and other escape hatches.
- Make illegal states unrepresentable: a union or enum instead of a loose string, a required field instead of an optional one that is always set.
- Validate external input (API, CLI, file, environment) once, at the boundary.

### Errors

- Fail loudly on programmer errors; handle expected failures explicitly.
- Do not swallow errors. Handle them, add context and rethrow, or let them propagate.
- Context means what was being attempted and on which identifier, never secrets or personal data.
- Follow the codebase's mechanism (exceptions, `Result` types, error returns) and release resources reliably (`finally`, `with`, `defer`).

### Tests

- Test behavior through the public interface, so tests survive refactoring.
- Name tests after the behavior: `returns_empty_list_when_no_orders_exist`.
- Cover the edges that matter for this change: empty, boundary, invalid input, the error path.
- Keep tests deterministic: control time, randomness and network at the boundary. Mock only what you have to; a test that mocks everything tests the mocks.

### Security basics

- No hardcoded secrets or credentials.
- Never build SQL, shell commands or HTML by concatenating user input; use parameters and escaping.
- Do not log passwords, tokens or personal data.
- Prefer the standard library or an existing dependency to adding a new one, and say so when you do add one.

### Performance

Correct first, clear second, fast only where measured. Avoid obvious waste (a query per loop iteration, loading a whole file to read one line), but do not trade readability for speed without evidence.

## Examples

**Over-engineered versus sufficient.** The request: "retry the upload up to 3 times".

```python
# Too much: a strategy class, a config object and a decorator for one call site
class RetryPolicy: ...
class ExponentialBackoff(RetryPolicy): ...
@with_retry(policy=ExponentialBackoff(RetryConfig(max_attempts=3)))
def upload(file): ...

# Enough
MAX_UPLOAD_ATTEMPTS = 3

def upload_with_retry(file):
    for attempt in range(1, MAX_UPLOAD_ATTEMPTS + 1):
        try:
            return upload(file)
        except TransientUploadError:
            if attempt == MAX_UPLOAD_ATTEMPTS:
                raise
```

**Comments and defensive clutter.**

```ts
// Before
// Get the user's discount
function getDiscount(user: User): number {
  try {
    if (user && user.plan) {            // User and plan are non-nullable types
      if (user.plan === "pro") {
        return 0.2; // return 20%
      }
    }
    return 0;
  } catch (e) {
    return 0;                           // hides any real bug as "no discount"
  }
}

// After
const PRO_DISCOUNT = 0.2;

function getDiscount(user: User): number {
  return user.plan === "pro" ? PRO_DISCOUNT : 0;
}
```

The function keeps its name, because renaming it would touch every caller for no requested reason. Dropping the guards is a cleanup only because the types rule out the cases they covered. If a caller could really pass a missing user, removing the `catch` would change behavior: fix that at the boundary and call it out, do not fold it into a tidy-up.

## Refactoring

1. Make sure tests pin down the current behavior. If none exist, add characterization tests first, because without them you cannot tell a refactor from a regression.
2. Move in small behavior-preserving steps and run the tests after each.
3. Keep refactoring separate from behavior changes, in separate commits where you are committing, so each can be reviewed and reverted on its own.
4. Stop when the code is clear enough for the task at hand. A refactor has no natural end, so the request sets it.

## Reviewing code

Review the change that was made, not the code you would have written. Group findings by severity so the author knows what blocks the merge:

- **Must fix:** bugs, security problems, data loss, broken tests.
- **Should fix:** maintainability problems, missing tests for new behavior, design problems the change introduces (speculative abstraction, a duplicated rule, a subtype that breaks its contract).
- **Nice to have:** naming and style polish. Keep this short; skip anything a formatter would settle.

For each finding give the location (`file:line`), the concrete impact, and a suggested fix. Say what is good when something is, and say so plainly if the change is fine as it is.

A review changes nothing, so the checklist under "Before finishing" does not apply to it. Run the tests on the change if you can, and say so if you could not; do not report a failure or a pass you did not see.

## Before finishing

This applies when you changed code. Run the project's formatter, linter and tests on what you changed. If the formatter rewrites lines you did not touch (the file was never formatted), keep its output on your lines only and mention that the file is unformatted. Then read your own diff from top to bottom as a reviewer would:

- [ ] Every changed line traces back to the request.
- [ ] It reads like the code around it in style and structure, without copying its defects or clutter.
- [ ] Nothing from "Habits to catch" is left in: no speculative abstraction, noise comments, needless guards, compatibility shims, debug output or dead code.
- [ ] New behavior is tested; no test was weakened to pass.
- [ ] Formatter, linter and tests were actually run and are clean.

Then report briefly: what changed, what you ran and its result, any trade-off you made, and anything you noticed but left out of scope.
