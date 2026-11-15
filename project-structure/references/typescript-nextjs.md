# TypeScript / Next.js

Check which router and which Next.js version the project uses before anything else (`app/` or `pages/`, and the `next` version in `package.json`). File conventions have changed between major versions, so confirm details against the docs for the installed version when unsure. This reference assumes the App Router.

## Baseline layout

```
my-app/
├── package.json
├── next.config.ts
├── tsconfig.json                 # "paths": { "@/*": ["./src/*"] }
├── public/                       # static files served as-is
├── e2e/                          # Playwright or Cypress
└── src/
    ├── app/                      # routing only
    │   ├── layout.tsx
    │   ├── page.tsx
    │   ├── (marketing)/
    │   ├── (app)/
    │   │   └── orders/
    │   │       ├── page.tsx
    │   │       ├── loading.tsx
    │   │       ├── error.tsx
    │   │       └── [orderId]/page.tsx
    │   └── api/.../route.ts
    ├── features/
    │   └── orders/
    │       ├── components/
    │       ├── actions.ts        # server actions ("use server")
    │       ├── queries.ts        # server-side data access
    │       ├── schemas.ts        # validation schemas and derived types
    │       └── index.ts          # optional: re-exports the feature's shared components and types
    ├── components/
    │   └── ui/                   # shared, feature-agnostic primitives
    ├── lib/                      # infrastructure clients: db, auth, logger
    └── env.ts                    # validated environment variables
```

- The `src/` directory is optional in Next.js. It keeps application code apart from the configuration files at the root. In an existing project, keep whichever arrangement it has.
- Start new projects with `create-next-app` and adjust.
- The root-level request interceptor file is `middleware.ts` in Next.js 15 and earlier and `proxy.ts` from Next.js 16. It sits beside `app/` (inside `src/` if that is used).

## Routing files stay thin

Inside `app/`, a folder becomes a route only when it contains `page.tsx` or `route.ts`, so the directory tree is the URL map. Keep it readable as one:

- `page.tsx` and `layout.tsx` fetch data, compose feature components and export metadata. Business logic and large component trees live in `features/`.
- Route groups `(name)/` organize routes and share layouts without affecting the URL.
- Private folders `_name/` are excluded from routing. Use them for pieces that are tied to exactly one route and have no meaning outside it (a page-specific layout section), colocated with it. A component that belongs to a product area goes in that feature, even if only one route uses it today.
- Use the special files (`loading`, `error`, `not-found`, `template`, `default`) for what they are for, in preference to hand-rolled equivalents.

Next.js supports three placements for non-route code: everything outside `app/`, top-level folders inside `app/`, or split per route. The layout above combines the first with route-level colocation for single-use pieces. Any of them is acceptable if applied consistently; follow what the project does.

## Features

A feature directory holds everything for one area of the product. Start it flat (a few files), and add `components/`, `hooks/` and so on when there are enough files to need them.

- A component used by one feature lives in that feature. It moves to `components/` when a second feature needs it and it is truly generic. If it carries domain meaning, the features probably share a concept that deserves its own feature directory.
- Features import from `components/`, `lib/` and, sparingly, from other features' public surface. `components/ui` and `lib/` never import from `features/`, and nothing imports from `app/`.
- `lib/` is conventional in Next.js projects and is the usual junk drawer. Keep it to infrastructure setup with one clearly named file or folder per concern (`lib/db.ts`, `lib/auth.ts`). Domain logic belongs in a feature.

**Barrel files.** One small `index.ts` per feature, re-exporting the components and types other features may use, is a reasonable boundary. Barrels at every level are not: they create import cycles, slow down tooling, and can pull server-only code into client bundles. Don't re-export client and server modules from the same barrel. A feature's server modules (`actions.ts`, `queries.ts`) therefore stay out of `index.ts` and are imported by their own path (`@/features/orders/queries`); those role files, together with `index.ts`, are the feature's public surface, and everything else in the feature is internal.

**Enforcing boundaries.** ESLint `no-restricted-imports` patterns, `eslint-plugin-boundaries`, or dependency-cruiser can forbid deep imports into another feature and imports in the wrong direction.

## Server and client code

The server/client split is part of the structure, because a misplaced file changes what ships to the browser.

- Components are server components by default. Put `"use client"` on the smallest interactive leaf, not on a page or layout, since everything a client component imports joins the client bundle.
- Server actions live in files that begin with `"use server"` (for example `actions.ts`), per feature.
- Modules that must never reach the client (database access, secrets) import the `server-only` package so that a wrong import fails the build.
- Read environment variables in one validated module. Only `NEXT_PUBLIC_*` variables are available in the browser. `.env.local` is not committed; commit a `.env.example`.

## Files and naming

- Next.js's special files have fixed lowercase names.
- For everything else follow the project's existing convention. Common ones are `kebab-case.tsx` for all files, or `PascalCase.tsx` for components with `camelCase.ts` for other modules. Mixing them causes trouble on case-insensitive filesystems.
- One exported component per file, named for the component. Small subcomponents used only by it can stay in the same file unexported; give one its own file when a second component needs it. Hooks are `use-thing.ts` or `useThing.ts`.
- A feature's role files (`actions.ts`, `queries.ts`, `schemas.ts`) start as one file each. Split one by sub-area (`queries/order-list.ts`) when it holds unrelated groups, not when it reaches a line count.
- Types live beside the code they describe. A feature-level `types.ts` is for types that several of the feature's files share; a global `types/` directory is a junk drawer by another name.
- Generated code (Prisma client, GraphQL or OpenAPI codegen) is written to its generator's output directory or a folder named `generated/`, excluded from linting, and never edited by hand.
- Dynamic segment names say what they are (`[orderId]`, not `[id]`), since they become the param names in code.

## Tests

- Unit and component tests are colocated: `order-summary.test.tsx` beside `order-summary.tsx`. Some projects use `__tests__/` folders instead; follow what is there.
- End-to-end tests live in a top-level `e2e/` (or `tests/`) directory with their own configuration.
- Shared test utilities and mocks go in one named place such as `src/test/`.
- `create-next-app` sets up no test runner. In an existing project use the one that is there. In a new one, Vitest (or Jest) with Testing Library for unit and component tests and Playwright for end-to-end are the usual choices; say which you added.

## Pages Router and plain TypeScript

- **Pages Router**: every file under `pages/` is a route, so nothing else may live there. Components and logic go in `features/`, `components/` and `lib/` as above. Don't mix in the App Router unless the user is migrating.
- **Libraries and Node services**: `src/` for source, build output in `dist/` (ignored), a small explicit public API in `src/index.ts`, and the `exports` field in `package.json` to define what consumers can import. Colocated tests excluded from the build.

## Monorepos

Use a single app until there is a second deployable or a package that is really shared. Then the conventional shape, with pnpm/npm/yarn workspaces and optionally Turborepo or Nx:

```
repo/
├── apps/
│   ├── web/
│   └── admin/
└── packages/
    ├── ui/
    ├── db/
    └── config/          # shared tsconfig, eslint config
```

Apps depend on packages; packages never depend on apps. Extract a package when two apps need the code, not in anticipation.

## Common problems to look for in an audit

- Business logic and data access written directly in `page.tsx`, `layout.tsx` or `route.ts`.
- A flat `components/` directory with dozens of files mixing generic primitives and feature-specific components.
- `lib/` or `utils/` holding domain logic, or a single large `utils.ts`.
- `"use client"` at the top of pages and layouts, or on files that don't need it.
- Server-only modules imported from client components, with no `server-only` guard.
- Deep relative imports (`../../../`) where a path alias exists, or deep imports into another feature's internals.
- Barrel files at every level, with import cycles.
- `process.env` read throughout the codebase.
- The same component or type defined in several features.
- Inconsistent file naming across the project.
- Files exporting several unrelated components, or a single `types.ts` or `constants.ts` for the whole application.
- Generated code edited by hand or mixed in with hand-written modules.
