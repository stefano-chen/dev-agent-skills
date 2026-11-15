---
name: project-structure
description: "Guidance for deciding where code lives: directory layout, module and package boundaries, file placement, file boundaries and naming. Use when scaffolding a new project or service, when a task creates new files, modules or packages in an existing codebase (to decide where they go, or whether the code belongs in an existing file instead), when moving, renaming, splitting or merging files, and when the user asks to review, audit, clean up or reorganize a project's structure, folders or architecture layout, even if they never say \"project structure\". Has specific references for Java/Spring, Python and TypeScript/Next.js; the principles apply to any stack. Not needed when the change clearly belongs in files that already exist."
argument-hint: "[what to scaffold, place or audit]"
---

# Project Structure

A project's structure is good when someone can guess where a piece of code lives before looking, and when a typical change touches few directories. Everything below serves those two tests. This is guidance, not a rulebook: where a principle doesn't fit the project in front of you, say why and do the sensible thing.

The scope is where code lives: directories, modules, and files as units (what a file holds, what it is called, when it splits). How code is arranged inside a file is code style, not structure; there, match the neighbouring files.

If text follows `/project-structure`, that text is the task.

## What wins when goals conflict

1. **What the user and the project say.** Explicit instructions, `CLAUDE.md`, `CONTRIBUTING.md`, ADRs and architecture docs.
2. **The structure the project already has.** A mediocre layout followed uniformly is easier to work in than a better one applied to half the code.
3. **The ecosystem's conventions.** Build tools, frameworks and newcomers all expect them.
4. **The principles below.**

## Pick the mode

- **Scaffold**: there is no project yet, or a new deployable/package is being added to a monorepo.
- **Place**: the project exists and the task adds or moves code. This is the common case, and it usually happens inside a larger task (a feature, a bug fix) rather than as a request about structure.
- **Restructure**: the user asked for a specific move, rename, split or merge ("split this file", "move billing into `features/`"). The decision is made; carry it out.
- **Audit**: the user asked for an opinion on the structure, or for an open-ended reorganization ("clean up the layout") where what should move is still to be decided.

Then, if the stack is one of these, read its reference before deciding anything. They hold the concrete layouts and the ecosystem-specific traps. A repository with several stacks needs one reference per stack. The TypeScript reference is written for Next.js: for another TypeScript stack (Vite, NestJS, plain Node) use its general parts (files and naming, tests, monorepos) together with that framework's own conventions.

- Java / Spring Boot: `references/java-spring.md`
- Python: `references/python.md`
- TypeScript / Next.js: `references/typescript-nextjs.md`

For any other stack, find its conventional layout (official docs, the framework's generator, well-known projects) and apply the principles to it.

## Principles

These are defaults for the decisions the project has not already made, and how much they decide depends on the mode. In scaffold mode they apply in full. In place mode they settle only what the existing structure leaves open: a layered project gets its new code in the layers, and a project that keeps shared helpers in `utils/` gets a truly shared helper there. In restructure mode they settle only what the request leaves open. In audit mode they are what the structure is measured against, and a deviation is a finding only when it is costing something.

**Ecosystem conventions first.** Where the build tool or framework defines a layout (Maven's `src/main/java`, Next.js's `app/`, Go's `internal/`), use it as given. Deviating costs tooling support and surprises every reader, and it is rarely worth what it buys.

**Group by feature once there are features to group.** Top-level directories named for what the system does (`billing/`, `search/`) keep a change inside one directory, because changes usually follow features. Directories named for technical roles (`controllers/`, `services/`, `models/`) spread every change across all of them. Layers still have a place inside a feature. A small app, a library or a script is often clearer flat or layered; feature-first starts paying when several features each have several files.

**Colocate what changes together.** Types, fixtures, styles and (where the ecosystem does it) tests sit beside the code they serve. Only what is truly cross-cutting, such as end-to-end tests, goes at the top level.

**Dependencies point one way.** Domain logic does not import infrastructure or framework glue. Features do not reach into each other's internals; they go through a small public surface. No cycles between modules. Prefer boundaries the tooling can enforce (visibility modifiers, lint import rules, architecture tests) over ones that depend on everyone remembering.

**Structure proportional to size.** Start flat. Split a directory when it has become hard to scan or holds clearly separate concerns, not before. Empty placeholder directories, single-file directories and deep nesting cost navigation effort now in exchange for a future that may not come.

**One responsibility per file.** A file holds one main thing (a type, a component, a cohesive set of functions) plus whatever exists only to support it, and its name says what that thing is. If the honest name needs an "and", or you would name it `misc`, it is two files. Split a file when it holds parts that change for different reasons or that different callers import separately. Length alone is a prompt to look, not a reason to split: a long file with one job is easier to read than the same code scattered over five. The opposite failure is as real: a file per tiny helper or per type, where reading one feature means opening ten tabs. How much goes in one file is partly an ecosystem convention (one top-level class per file in Java, several related classes per module in Python), so check the reference.

**No junk drawers.** `utils/`, `common/`, `helpers/`, `shared/` and `misc/` attract unrelated code because nothing visibly doesn't belong there. Name a module for what it holds (`dates`, `money`, `http_client`). Leave a helper beside its one caller. When a caller in another module needs it, move it to a module named for what it does, so that nobody copies it.

**Clean root, thin entry points.** Source, configuration, scripts, docs and generated output are visibly separate, and build output is ignored by version control. Generated code (API clients, ORM classes, codegen output) goes where its generator puts it or in a directory named as generated, is never edited by hand, and is not mixed into hand-written directories; whether it is committed follows what the project already does. `main` and its equivalents wire things together and hold no logic of their own, so the logic stays testable.

**Consistent naming.** One case style per kind of thing, following the language's convention. The same concept gets the same name everywhere, and a file is named for its main export or responsibility.

## Scaffold mode

1. Establish what is being built (web app, API, library, CLI, a mix), the stack, and the expected scale. If any of these is unknown and would change the layout, ask once. Otherwise state your assumption and go on.
2. Use the ecosystem's own generator when there is one (Spring Initializr, `create-next-app`, `uv init`), then adjust. The output matches what current tooling expects, which a hand-written tree may not. Run it non-interactively (pass the choices as flags) and into an empty target directory. If it cannot run (no network, a directory that already has content), write the same layout by hand from the reference and say so. Afterwards remove the demo content the project will not use (sample pages, placeholder assets), and if the generator set up no test runner, add the usual one for the stack and say which. In a monorepo the sibling packages are the template instead: match their layout, tooling and configuration, and register the new package wherever they are registered (workspace manifest, CI, build config).
3. Create only what the first working version needs: an entry point, one real feature or module, a test that runs, configuration, a README with run instructions, and a `.gitignore`. A directory appears when its first file does.
4. Decide the things that are expensive to change later, and tell the user what you chose and why: the root package or module name, where tests live, how configuration and secrets are loaded, and single package versus workspace.
5. Verify that it builds and the test runs, then show the resulting tree.

## Place mode

The aim is for new code to look as if the project's own authors had put it there.

Most of this is silent. A placement that follows the project's pattern needs no explanation. Tell the user only what they would want a say in: a new top-level directory or shared module, a split between two patterns, a placement that had no analogue, or a structural problem you worked around.

1. **Infer the structure before adding to it.** List the top-level directories, then find the nearest analogue of what you are adding (an existing endpoint, entity, component or job) and note where each of its parts lives, how the files are named, and where its tests are.
2. **Follow that pattern**, even where you would have designed it differently. If the project mixes two patterns, match the module you are working in. Where that doesn't settle it, look for a migration under way (an ADR, recent commits moving code in one direction) and follow its direction; failing that, follow the most recent code. Mention the split to the user. Check that the analogue is typical before copying it: if its siblings are laid out differently, it is the outlier, and the siblings are the pattern.
3. **Decide between a new file and an existing one.** New code goes into an existing file when it is part of that file's responsibility, and the analogue shows how much the project puts in one file. It gets a new file when it is a separate thing by the project's own granularity (a new endpoint where each endpoint has a file), or when adding it would give the existing file a second job. If the file you are editing has already outgrown its responsibility, adding the new part as its own file is fine; splitting up what is already there is restructuring, covered by step 6.
4. **Without an analogue**, apply the principles in the least surprising way: the narrowest scope that works, beside its consumer, named like its neighbours.
5. **Before creating a new top-level directory or a new shared module**, check whether an existing one is already meant for this. A new top-level entry changes the map everyone uses, so say that you are adding one and why.
6. **Don't restructure along the way.** If the existing structure is getting in the way of the task, finish the task within it and report the problem separately, with a suggestion. Moving files inside an unrelated change makes the diff hard to review, breaks other people's open branches, and is a decision that belongs to the user.
7. **Check what the placement touches.** A new or moved file can need registering somewhere: a barrel or `__init__` re-export, a router or module registration, build or test globs, path-based ownership or CI rules. Look at how the analogue is wired in, do the same, then run the build and the tests. When the task itself is a move or rename, use the version-control move command and search for the old path in imports, configuration, docs and scripts.

## Restructure mode

The user has already decided what moves, so do not turn the request into an audit or wait for a second approval.

1. Settle only what the request leaves open (the new name, the exact destination, where a split falls), using the nearest analogue as in place mode. Ask only if the options differ in a way the user would care about.
2. Carry it out with the version-control move command so history follows, keep the move separate from behavior changes, and do the wiring check from place mode step 7: registrations, re-exports, globs, and every reference to the old path in imports, configuration, docs and scripts. Run the build and the tests.
3. Keep to what was asked. Other structural problems you notice go in the report as suggestions.

## Audit mode

1. **Map what exists**: the top-level layout, the main modules and how they depend on each other, where tests and configuration live, and which conventions are followed consistently.
2. **Find what the structure is costing.** Evidence is better than taste, and it comes from version control (`git log --name-only` shows which files change together and which change in most commits) and from the stack's dependency and lint tooling, more than from reading the tree. Look for: recent commits that each touch many directories, import cycles, a module everything depends on, junk-drawer directories that keep growing, the same concept in several places, deep nesting, directories with one file, files that hold several unrelated things and show up in most commits, features shredded into many few-line files, entry points full of logic, generated code edited by hand or mixed in with hand-written code, deviations from the ecosystem layout that tooling has to be configured around.
3. **Report**, most costly first. For each finding: what it is (with paths), what it costs, and the suggested change. Say what is working well too, so it survives the reorganization. Keep clear which findings are problems and which are only preferences, and leave preferences out unless asked.
4. **Propose an incremental migration**, not a rewrite: small steps that each leave the build and tests passing, in an order where the early steps are useful even if the later ones never happen. Note what each step touches outside the source tree (imports, build config, CI paths, docs, deployment manifests).
5. **Change nothing until the user picks what to do.** When executing, move files with the version-control move command so history follows, keep moves separate from behavior changes, and run the build and tests after each step.

Judge the structure against the project's actual size and team. A flat layout in a 15-file service is not a finding.
