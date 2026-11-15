---
name: design-interview
description: "Pre-implementation workflow for a new feature or a bug fix. Explores the codebase, then interviews the user on design decisions (feature) or investigates the root cause (bug). Writes no code until the user approves."
argument-hint: "[feature or bug description]"
disable-model-invocation: true
model: claude-fable-5-2
---

# Design Interview

A pre-implementation workflow. Its output is shared understanding, not code. The user invoked this because they want to settle the design (or the diagnosis) before anything changes, so until they approve the result:

- **Do not create, edit or scaffold anything in the source tree**, not even a stub or a failing test.
- **Read and search freely.** Run read-only commands. Run the existing tests and the program when what they leave behind is local and disposable (build output, caches, a test database the suite creates for itself).
- **Ask before anything whose effects outlast the run or reach beyond local dev.** That means writing to a real or shared database, running migrations, sending anything to an external service, installing or upgrading dependencies, and changing the working tree or git state (checkout, stash, reset, bisect). Say what it would change and why you need it. If you cannot tell whether running something is safe, treat it as not safe.
- **When the investigation itself needs a change** (a bug that only shows on a write, a temporary log line, `git bisect`), do it on a throwaway copy of the repository in a temp directory where that is practical, and delete the copy afterwards. Otherwise ask.

Approval means an explicit go-ahead on the finished brief or report. Answering an interview question, or agreeing with a recommendation along the way, is not approval to build.

Request: `$ARGUMENTS`. If that is empty, or shows up as a literal placeholder, ask the user for it in one sentence.

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
- Tests covering the area, and any docs/ADRs

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
- **UX**: where it appears, states (loading/empty/error), copy, accessibility
- **Tradeoffs**: performance vs. simplicity, build vs. reuse, flexibility vs. YAGNI, rollout strategy

Rules for every question:

1. **Never ask what the code can answer.** If you could find it by reading or searching, go find it.
2. **Don't ask what the codebase has already settled.** Where an established convention decides the matter, follow it and record it under **Assumptions** so the user can veto it. Ask only when conventions conflict or the choice is a product decision.
3. **Always offer a recommended default** with a one-line reason, grounded in what you saw in the codebase where possible. Format:
   > **Q:** How should duplicate entries be handled?
   > **Recommended:** Reject with a validation error, matching how `UserService.create` handles conflicts.
   > Alternatives: silently merge; allow duplicates.
4. **Ask in small batches** (1-3 related questions per turn), highest-impact decisions first. When the `AskUserQuestion` tool is available, use it for questions with a few distinct options, with the recommended option listed first and labeled "(Recommended)". The dialog can hide what you wrote before it, so each question must make sense on its own. Ask open-ended questions in plain text.
5. If the user answers "default" / "you pick" / doesn't care, take the recommendation and move on.
6. **Flag conflicts instead of absorbing them.** If an answer contradicts the code, an earlier decision, or a constraint you found, say what it would break or cost and ask which one wins.
7. Let answers reshape later questions: drop ones that became moot, add ones newly opened.
8. **Don't ask the user to guess at facts.** If a decision hinges on something nobody in the conversation knows (how a library behaves, whether a query is fast enough), check it in the docs or with a read-only experiment in a temp directory outside the source tree. If it can't be checked now, record it as a risk in the brief.
9. Keep a running **Decisions log**, each entry with its reason, and show it briefly every two or three rounds so the user can correct drift.
10. If the user cuts the interview short ("enough, just build it"), turn every remaining question into an assumption using your recommendation, list them in the brief, and get the go-ahead on that.

### Step 4: Know when to stop

Keep asking until an engineer could implement it without making a product or design decision on their own. Stop when every open item is either decided or explicitly deferred. Don't pad with low-value questions.

### Step 5: Deliver the brief

End with an **Implementation brief**. Implementation may happen in a fresh session, so write it for a reader who has not seen this conversation: full paths, no "as discussed above". Scale it to the task and leave out any section that has nothing in it.

- **Goal**: one or two sentences, including the problem being solved
- **Scope**: in / out
- **Decisions**: the full log, each with the chosen option, the reason, and the alternatives that were turned down
- **Assumptions**: what you took from the codebase without asking
- **Edge cases and handling**
- **Data / schema changes**
- **Affected files and components** (paths)
- **Build order**: the steps in the sequence you would take them
- **Acceptance criteria**: observable checks that show it is done
- **Test plan**: the cases that must be covered
- **Risks, open questions and deferred items**
- **How to implement**: the rules under "After the user approves", copied in, so they reach a session that never loaded this skill

Then ask two separate things, in this order: whether to save the brief as a markdown file (see "Saving the result"), and whether to proceed to implementation. A reply that clearly answers one settles only that one. If a reply could be answering either, it is not approval to build: ask. Do not start coding until the user says so.

---

## Bug mode

### Step 1: Explore and reproduce

- Find the code path behind the reported behavior.
- Try to reproduce: run the relevant tests or the program. If you need a script to exercise existing code, put it in a temp directory outside the source tree. It must only call existing code, never patch it. Keep it until the fix has been verified against it, or until the user decides not to proceed, then remove it. Include its content or the exact commands in the report so the reproduction can be rebuilt in another session.
- Check logs and recent changes in the area (`git log`, `git blame`).
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
- **Regression test**: the case to add (described, not written)
- **Confidence**: high / medium / low, and what would raise it. If you never reproduced the bug, confidence is medium at most.
- **How to implement**: the rules under "After the user approves", copied in

If the evidence did not single out one cause, say so at the top. **Root cause** becomes the ranked candidates, and **Fix options** becomes the next diagnostic step: what to observe, what that needs (temporary logging, access to data or an environment), and what each outcome would mean. Ask for the go-ahead on that step, not on a fix.

Then ask two separate things, in this order: whether to save the report (see "Saving the result"), and which fix option to proceed with, if any. A reply that could be answering either is not approval to build: ask. Do not write the fix until the user says so.

---

## Saving the result

A brief or report that exists only in the conversation is lost when the session ends or is cleared, and can be thinned by compaction. After delivering it, offer to save it as a markdown file where the repo keeps specs or design docs (ask where if there is no obvious place). This is the one file you may write before approval, and only when the user says yes.

## After the user approves

The brief or report is now the contract. It decides what gets built. Other skills active during implementation (test-driven development, code quality, project structure) decide how: their process and style rules apply, the brief's test plan is the starting test list, and where they would have you pick an interpretation yourself, the first rule below wins. Give one final report covering all of them.

- **Feature**: follow the build order. If you meet a decision the brief doesn't cover, stop and ask rather than choosing; then add the answer to the decisions log.
- **Bug**: write the regression test first and watch it fail, then make the fix, then confirm the test passes and the original reproduction no longer shows the symptom.
- Finish by running the test plan and checking each acceptance criterion (or each point of the report) against what you observed, not against what the code should do. Tell the user anything that deviated, and anything you could not verify, and why.
