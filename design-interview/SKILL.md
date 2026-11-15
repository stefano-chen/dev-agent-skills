---
name: design-interview
description: "Pre-implementation workflow for a new feature or a bug fix. Explores the codebase, then interviews the user on design decisions (feature) or investigates the root cause (bug). Writes no code until the user approves. Run it only when the user asks for it by name; do not start it on your own."
argument-hint: "[feature or bug description]"
disable-model-invocation: true
---

# Design Interview

A pre-implementation workflow. Its output is shared understanding, not code. The user invoked this because they want to settle the design (or the diagnosis) before anything changes, so until they approve the result:

- **Do not create, edit or scaffold anything in the source tree**, not even a stub or a failing test.
- **Read and search freely.** Run read-only commands. Run the existing tests and the program when what they leave behind is local and disposable (build output, caches, a test database the suite creates for itself).
- **Ask before anything whose effects outlast the run or reach beyond local dev.** That means writing to a real or shared database, running migrations, sending anything to an external service, installing or upgrading dependencies, and changing the working tree or git state (checkout, stash, reset, bisect). Say what it would change and why you need it. If you cannot tell whether running something is safe, treat it as not safe.
- **When the investigation itself needs a change** (a bug that only shows on a write, a temporary log line, `git bisect`), do it on a throwaway copy of the repository in a temp directory where that is practical, and delete the copy afterwards. Otherwise ask.

Approval means an explicit go-ahead on the finished brief or report. Answering an interview question, or agreeing with a recommendation along the way, is not approval to build.

The request is whatever the user gave when they invoked this skill: the text that came with the invocation, or else the message that asked for it. If there is none, ask the user for it in one sentence.

If the request points at a brief or report saved by an earlier run, do not start over. Read it, check that it still matches the code (the paths exist, the behavior it cites is unchanged), raise anything that has gone stale, then ask for the go-ahead and continue under "After the user approves".

## 0. Pick the mode and size the effort

- **Feature mode**: the user wants to add or change behavior ("add X", "support Y", "I want Z").
- **Bug mode**: something happens that shouldn't ("X happens instead of Y", "crash when...", "broken").

Work that should change no behavior (a refactor, a migration, performance work) goes through feature mode. The interview is then about what must stay the same and how that will be verified.

If it is unclear, ask once. Then restate the request in one line with the mode you picked, so a misreading surfaces now, and start.

Size the process to the task. A small, local change needs a short exploration and perhaps two questions; a change that crosses modules or touches stored data needs the full treatment. Don't run a long interview for a one-line change, and don't shortcut a risky one.

If the work shows the other mode fits better (the "bug" is behavior nobody built, or the "feature" is existing behavior that is broken), say so in one line and switch.

---

## Feature mode

### Step 1: Explore first

Before asking anything, explore the parts of the codebase the feature will touch:

- Entry points, modules and data models involved
- Existing patterns for similar features (follow the house style, don't invent one)
- Config, feature flags, permissions, persistence/schema, APIs, UI components
- Tests covering the area and how they are run (framework, command), or the fact that the project has no test setup. Note which of the code the feature will change has no test covering it, because the build order needs to know.
- Any docs/ADRs

If there is no codebase yet (a new project, or a new package in a monorepo), there is little to explore. Look at what surrounds it (sibling packages, the repository's tooling and conventions), and expect the setup decisions in Step 3 to carry more of the interview.

Write a short **Context summary** for the user: what exists today, where the feature would plug in, and the constraints you found. Cite file paths. Three bullets is fine for a small change; stay under ten.

### Step 2: Check the premise

Before interviewing, tell the user if exploration turned up any of these, and let them decide how to continue:

- The feature, or most of it, already exists.
- It conflicts with existing behavior or a documented decision.
- A much smaller change would meet the stated goal.

Also make sure you know the goal behind the request: what problem it solves and for whom. If the request names only a solution ("add a cache") and the reason isn't evident from the request or the code, ask for it before anything else. Every later tradeoff is judged against the goal, and a recommendation made without it is a guess.

### Step 3: Interview

Ask only about **decisions that affect the design**. Cover, as relevant:

- **Behavior**: what exactly happens, triggers, inputs/outputs, scope (what is explicitly out)
- **Edge cases**: empty/invalid input, concurrency, failures, limits, permissions, backwards compatibility
- **Data**: new fields/tables, migrations, validation, defaults, retention, source of truth
- **Side effects**: what the feature causes beyond its direct output (calls to external services, emails or notifications, emitted events or webhooks, cache invalidation, audit logs or metrics, writes to state other code reads), and what happens to each on retry, partial failure or rollback
- **UX**: where it appears, states (loading/empty/error), copy, accessibility
- **Tradeoffs**: performance vs. simplicity, build vs. reuse, flexibility vs. YAGNI, rollout strategy
- **Setup for the implementation**, only where it applies and the project has not already settled it. These are decisions the implementation would otherwise have to stop for, or make on its own:
  - *No test setup*: add one (and which runner), or verify by running the code. For a small change, recommend not adding one.
  - *New project or package*: the stack, the root package or module name, where tests live, how configuration and secrets are loaded, single package or workspace.
  - *Commits*: whether to commit during implementation, and at which points. Ask when the work has several steps, or mixes a move or refactor with a behavior change, because those are worth separate commits. Otherwise the default is no commits unless the user asks for them.

Rules for every question:

1. **Never ask what the code can answer.** If you could find it by reading or searching, go find it.
2. **Don't ask what the codebase has already settled.** Where an established convention decides the matter, follow it and record it under **Assumptions** so the user can veto it. Ask only when conventions conflict or the choice is a product decision.
3. **Always offer a recommended default** with a one-line reason, grounded in what you saw in the codebase where possible. Format:
   > **Q:** How should duplicate entries be handled?
   > **Recommended:** Reject with a validation error, matching how `UserService.create` handles conflicts.
   > Alternatives: silently merge; allow duplicates.
4. **Ask in small batches** (1-3 related questions per turn), highest-impact decisions first. If the environment gives you a structured way to ask multiple-choice questions (a question dialog or form), use it for questions with a few distinct options, with the recommended option listed first and labeled "(Recommended)". Such a dialog can hide what you wrote before it, so each question must make sense on its own. Ask open-ended questions, and every question where there is no such dialog, in plain text.
5. If the user answers "default" / "you pick" / doesn't care, take the recommendation and move on.
6. **Flag conflicts instead of absorbing them.** If an answer contradicts the code, an earlier decision, or a constraint you found, say what it would break or cost and ask which one wins.
7. Let answers reshape later questions: drop ones that became moot, add ones newly opened.
8. **Don't ask the user to guess at facts.** If a decision hinges on something nobody in the conversation knows (how a library behaves, whether a query is fast enough), check it in the docs or with a read-only experiment in a temp directory outside the source tree. If it can't be checked now, record it as a risk in the brief.
9. Keep a running **Decisions log**, each entry with its reason, and show it briefly every two or three rounds so the user can correct drift.
10. If the user cuts the interview short ("enough, just build it"), turn every remaining question into an assumption using your recommendation, list them in the brief, and get the go-ahead on that.

### Step 4: Know when to stop

Keep asking until an engineer could implement it without making a product or design decision on their own. Stop when every open item is either decided or explicitly deferred. Don't pad with low-value questions.

Before you stop, draft the test plan and use it to find the decisions that are still missing. Implementation works through these cases one at a time, and a case whose expected result nobody decided comes back as a question in the middle of the build. For each behavior in scope, go through:

- the happy path
- edge cases: empty, zero, one, many, largest and smallest values, unicode, duplicates, ordering
- invalid input and error paths
- boundaries (`<` vs `<=`, first and last element)
- state and interaction: called twice, concurrent calls, idempotency
- side effects: each fires exactly when intended, not on a failed or rolled-back operation, and not twice on a retry

Keep the cases that apply. Where the expected result follows from a decision already made, or from how the codebase handles the same case, write it down. Where it does not, that is an interview question: ask it now, with a recommendation, under the rules of Step 3. For a small change this is a quick pass over a few cases, not a new round of questions.

### Step 5: Deliver the brief

End with an **Implementation brief**. Implementation may happen in a fresh session, so write it for a reader who has not seen this conversation: full paths, no "as discussed above". Scale it to the task and leave out any section that has nothing in it.

- **Goal**: one or two sentences, including the problem being solved
- **Scope**: in / out
- **Decisions**: the full log, each with the chosen option, the reason, and the alternatives that were turned down
- **Assumptions**: what you took from the codebase without asking
- **Edge cases and handling**
- **Data / schema changes**
- **Side effects**: each one, when it fires, and how it behaves on retry, failure or rollback
- **Affected files and components** (paths): the existing files to change, and the new files, marked as new. Choose where a new file goes by the project's own pattern: the layout and naming of its nearest analogue.
- **Build order**: the steps in the sequence you would take them. Each step is one of three kinds and never mixes them, because a step that changes structure and behavior together cannot show which of the two broke a test:
  - *Behavior*: adds one piece of observable behavior end to end, cut as a thin vertical slice where the work allows, and names the test-plan cases that prove it. New code with nothing testable of its own (a schema change, new wiring) is joined to the first behavior step that uses it.
  - *Structural*: a refactor, move or rename of existing code that a later step needs, with no change in behavior. It names the existing tests that must pass before and after, and it comes before the behavior step it makes room for.
  - *Pin*: tests that record what existing code does today, added before a step that changes or restructures code no test covers. It names the behavior to pin. If the project has no test setup and none is being added, there is nothing to pin with: list the uncovered code under Risks instead.

  In work that changes no behavior, every step is structural or pin.
- **Acceptance criteria**: observable checks that show it is done
- **Test plan**: the cases that must be covered, each with its expected result
- **Implementation setup**: the test framework and command (or what was decided if the project has none), the commit policy, and for a new project or package the stack and layout choices
- **Risks, open questions and deferred items**
- **How to implement**: the rules under "After the user approves", copied in, so they reach an agent that never read this skill

Then ask two separate things, in this order: whether to save the brief as a markdown file (see "Saving the result"), and whether to proceed to implementation. A reply that clearly answers one settles only that one. If a reply could be answering either, it is not approval to build: ask. Do not start coding until the user says so.

---

## Bug mode

### Step 1: Explore and reproduce

- Find the code path behind the reported behavior.
- Try to reproduce: run the relevant tests or the program. If you need a script to exercise existing code, put it in a temp directory outside the source tree. It must only call existing code, never patch it. Keep it until the fix has been verified against it, or until the user decides not to proceed, then remove it. Include its content or the exact commands in the report so the reproduction can be rebuilt in another session.
- Check logs and recent changes in the area (`git log`, `git blame`).
- Note how the project's tests are run, or that it has no test setup. The regression test depends on it.
- Record what you ran and what you observed.

### Step 2: Find the root cause, not the symptom

Trace back from where the wrong behavior surfaces to where the wrong assumption or state is introduced. Keep asking "why does this happen?" until the answer is a concrete defect (bad logic, wrong assumption, race, missing validation, bad data, config, dependency change).

When more than one explanation fits the evidence, name each one and find the check that tells them apart, then run it. Confirm with evidence rather than reasoning alone wherever possible, and keep clear in your notes which claims you observed and which you inferred.

If the evidence does not single out one cause, don't present the likeliest guess as the cause. Report the candidates in order of likelihood and say what observation would settle it.

### Step 3: Ask only for what only the user has

If blocked, ask for information that isn't in the code or logs: exact repro steps, environment/versions, when it started, what the expected behavior really is, sample data. **One question at a time**, and say briefly why you need it. Never ask what you could find yourself.

### Step 4: Report (no fix yet)

- **Symptom**: what happens vs. what should happen
- **Reproduction**: steps/commands and result, or why it couldn't be reproduced
- **Root cause**: the defect, with file:line references
- **Evidence**: test output, logs, traces, or the code path that proves it, with observed and inferred kept apart
- **Introduced by**: the commit or change, if you found it
- **Why tests missed it**: the gap in coverage
- **Blast radius**: other code affected by the same cause or pattern
- **Fix options**: 1-3 approaches with tradeoffs and a recommendation (described, not written)
- **Regression test**: the case to add (described, not written). If the project has no test setup, say so and recommend one of two paths: add a setup (name the runner) and the test, or verify the fix against the reproduction only. For a small fix, recommend the second.
- **Confidence**: high / medium / low, and what would raise it. If you never reproduced the bug, confidence is medium at most.
- **How to implement**: the rules under "After the user approves", copied in

If the evidence did not single out one cause, say so at the top. **Root cause** becomes the ranked candidates, and **Fix options** becomes the next diagnostic step: what to observe, what that needs (temporary logging, access to data or an environment), and what each outcome would mean. Ask for the go-ahead on that step, not on a fix.

Then ask two separate things, in this order: whether to save the report (see "Saving the result"), and which fix option to proceed with, if any. Add two things to the second question when they apply:

- If **Blast radius** lists other sites, ask whether the fix should cover them too. The default is the reported bug only, with the rest left as follow-ups.
- If the bug was never reproduced, say that the fix cannot be shown to work until it is, so that picking an option is an informed choice of an unverified fix.

A reply that could be answering either question is not approval to build: ask. Do not write the fix until the user says so.

---

## Saving the result

A brief or report that exists only in the conversation is lost when the conversation ends, and can be thinned when a long conversation is summarized. After delivering it, offer to save it as a markdown file where the repo keeps specs or design docs (ask where if there is no obvious place). This is the one file you may write before approval, and only when the user says yes.

## After the user approves

The brief or report is now the contract. It decides what gets built. How the code is written (style, testing technique, file layout) follows the project's conventions and whatever coding guidance the session provides, such as the project's instruction files: if some of it applies to this work and you have not read it yet, read it before the first change. Whatever that guidance says, the brief's test plan is the starting list of tests, and its setup decisions (test setup, commits, layout) are settled, so do not ask them again. Give one final report for the whole task.

Two rules apply to every kind of work below:

- **Open decisions.** If you meet a product or design decision the brief or report doesn't cover (behavior a user can observe, stored data or schema, a public interface, a new dependency, scope), stop and ask rather than choosing, even where other guidance would have you pick an interpretation yourself. Then add the answer to the decisions log. Details an engineer settles without asking (a private name, the shape of a helper, internal structure) are yours: follow the codebase. If nobody is there to answer (an unattended run), take the option that is easiest to reverse, continue, and list it as an assumption in the final report. Stop only when every option is expensive to reverse.
- **File placement.** The paths the brief gives for new files are the planned placement, not a design decision. If the project's own pattern, as you find it when you get there, puts a file somewhere else, follow the pattern and report the difference. A new top-level directory or shared module that the brief did not name goes back to the user first.

Then, by kind of work:

- **Feature**: follow the build order one step at a time, and finish each step, with the tests passing, before starting the next. A behavior step is done when its test-plan cases pass. A structural step is done when the tests it names pass with their assertions untouched. A pin step is done when its new tests pass against code that has not been changed yet. If a step turns out to need a structural change or a pin that the build order did not plan, do that first as a step of its own and say so in the final report.
- **Bug**: write the regression test first and watch it fail, then make the fix, then confirm the test passes and the original reproduction no longer shows the symptom. Two cases change this:
  - *The bug was never reproduced.* Try to make a test fail at the level of the suspected defect. If it fails for the reason the report gives, that is the reproduction: continue as above. If nothing can be made to fail, the fix is a guess the user accepted when they picked it. Make it, keep the test as a guard for the expected behavior, and label the fix as unverified in the final report, with the observation that would confirm it.
  - *The project has no test setup.* Do what the report recommended, unless the user chose otherwise: add the runner it named and then the regression test, or verify the fix against the original reproduction and say in the final report that no regression test exists.
- **Diagnostic step** (the report could not single out a cause, and the user approved its next step): do that step and nothing else. Anything it adds to the source tree (a log line, a probe) is temporary. Remove all of it once the observation is made, and check the diff to confirm nothing is left. If it has to stay in place to collect evidence (logging that must be deployed), say so and list exactly what to remove later. Then go back to bug mode Step 2 with the new evidence and deliver an updated report. The approval covered the diagnosis; a fix needs its own.
- After a feature or a fix, finish by running the test plan and checking each acceptance criterion (or each point of the report) against what you observed, not against what the code should do. Tell the user anything that deviated, and anything you could not verify, and why.
