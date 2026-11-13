---
name: design-interview
description: "Pre-implementation workflow for a new feature or a bug fix. Explores the codebase, then interviews the user on design decisions (feature) or investigates the root cause (bug). Writes no code until the user approves."
argument-hint: "[feature or bug description]"
disable-model-invocation: true
---

# Design Interview

A pre-implementation workflow. Its output is shared understanding, not code. The user invoked this because they want to settle the design (or the diagnosis) before anything changes, so until they approve the result:

- **Do not create, edit or scaffold anything in the source tree**, not even a stub or a failing test.
- **Read and search freely.** Run existing tests, the program, and read-only commands.
- **Only run commands that are safe to repeat.** Nothing that writes data, runs migrations, installs or upgrades dependencies, changes the working tree or git state (checkout, stash, reset), or touches anything outside local dev. If you need one of these to make progress, ask first and say why.

Request: `$ARGUMENTS`. If that is empty, ask the user for it in one sentence.

## 0. Pick the mode and size the effort

- **Feature mode**: the user wants to add or change behavior ("add X", "support Y", "I want Z").
- **Bug mode**: something happens that shouldn't ("X happens instead of Y", "crash when...", "broken").

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
4. **Ask in small batches** (1-3 related questions per turn), highest-impact decisions first. When the `AskUserQuestion` tool is available, use it, with the recommended option listed first and labeled "(Recommended)".
5. If the user answers "default" / "you pick" / doesn't care, take the recommendation and move on.
6. **Flag conflicts instead of absorbing them.** If an answer contradicts the code, an earlier decision, or a constraint you found, say what it would break or cost and ask which one wins.
7. Let answers reshape later questions: drop ones that became moot, add ones newly opened.
8. Keep a running **Decisions log** and show it briefly every two or three rounds so the user can correct drift.

### Step 4: Know when to stop

Keep asking until an engineer could implement it without making a product or design decision on their own. Stop when every open item is either decided or explicitly deferred. Don't pad with low-value questions.

### Step 5: Deliver the brief

End with an **Implementation brief**:

- **Goal**: one or two sentences
- **Scope**: in / out
- **Decisions**: the full log, each with the chosen option
- **Assumptions**: what you took from the codebase without asking
- **Edge cases and handling**
- **Data / schema changes**
- **Affected files and components** (paths)
- **Build order**: the steps in the sequence you would take them
- **Acceptance criteria**: observable checks that show it is done
- **Test plan**: the cases that must be covered
- **Open questions / deferred items**

Then offer to save the brief as a markdown file (see "Saving the result") and ask whether to proceed to implementation. Do not start coding until the user says so.

---

## Bug mode

### Step 1: Explore and reproduce

- Find the code path behind the reported behavior.
- Try to reproduce: run the relevant tests or the program. If you need a script to exercise existing code, put it in a temp directory outside the source tree and remove it when done. It must only call existing code, never patch it.
- Check logs and recent changes in the area (`git log`, `git blame`).
- Record what you ran and what you observed.

### Step 2: Find the root cause, not the symptom

Trace back from where the wrong behavior surfaces to where the wrong assumption or state is introduced. Keep asking "why does this happen?" until the answer is a concrete defect (bad logic, wrong assumption, race, missing validation, bad data, config, dependency change).

When more than one explanation fits the evidence, name each one and find the check that tells them apart, then run it. Confirm with evidence rather than reasoning alone wherever possible, and keep clear in your notes which claims you observed and which you inferred.

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

Then offer to save the report (see "Saving the result") and ask whether to proceed with the fix. Do not write it until the user says so.

---

## Saving the result

A brief or report that exists only in the conversation is lost on `/clear` and can be thinned by compaction. After delivering it, offer to save it as a markdown file where the repo keeps specs or design docs (ask where if there is no obvious place). This is the one file you may write before approval, and only when the user says yes.

## After the user approves

The brief or report is now the contract.

- **Feature**: follow the build order. If you meet a decision the brief doesn't cover, stop and ask rather than choosing; then add the answer to the decisions log.
- **Bug**: write the regression test first and watch it fail, then make the fix, then confirm the test passes and the original reproduction no longer shows the symptom.
- Finish by checking the result against the acceptance criteria or the report, and tell the user anything that deviated and why.
