---
name: nestjs-module-migration
description:
  This skill covers migrating a legacy backend module into the NestJS monorepo,
  including shared contracts/auth/pipe patterns, keeping
  `MODULARISATION-STATUT.md` in sync, and preserving kanban task tracking.
license: MIT
metadata:
  hermes:
    tags: [nestjs, modularisation, migration, kanban]
---

# NestJS Module Migration

## Scope

Use this skill when migrating a legacy route/resource into
`apps/server-nestjs/src/modules/<name>`.

## Standard anatomy

- `controller.ts`: controllers, decorators, `@UseGuards(...)`,
  `ZodValidationPipe`
- `service.ts`: business logic; uses `PrismaService`
- `module.ts`: imports `InfrastructureModule` + `AdminModule` when admin auth is
  needed
- `<name>.spec.ts`: unit tests aligned with repo mocks

## Shared-contract rule

Mirror route shape/validation using `packages/shared/src/contracts/<domain>.ts`
and schemas in `packages/shared/src/schemas/<domain>.ts`. Prefer
`ZodValidationPipe` with the shared schema body/query types.

## Auth rule

- admin routes: `AdminGuard` + `@RequireAdminPermission(...)`
- project routes: `ProjectContextGuard`, `ProjectStatusGuard`,
  `ProjectLockedGuard`, `UserGuard`, plus `@Project()` /
  `@RequireProjectStatus()`
- keep path-level controller first, conditional bypass only when required

## MainModule wiring

- add new `*Module` to `apps/server-nestjs/src/main.module.ts`
- remove stale/broken module references after rebase/conflict resolution
- run `pnpm --filter server-nestjs exec tsc --noEmit` after edits

## Documentation + tracking

- update
  `apps/server-nestjs/documentation/Modularisation-de-console-server/MODULARISATION-STATUT.md`
  with:
  - module section (routes, permissions, files, legacy differences)
  - route totals and percentage progress
  - new key date entry
  - changelog entry
- if user requested task tracking, keep migration metadata in the task item
  (module, route counts, permissions, files)

## Legacy parity

- keep endpoint behavior as close as possible to
  `apps/server/src/resources/<module>/`
- preserve “Différences avec le legacy” notes for client/QA

## Pitfalls

- New Prisma service code must use the repo-standard
  `$transaction(async (tx) => { ... })` callback form; array-based
  `$transaction([...])` does not match the standard vitest mocks used elsewhere.
- After parallel rebases, `main.module.ts` can retain stale imports for modules
  that were renamed/removed on the rebased side; always reconcile imports
  against actual files in `apps/server-nestjs/src/modules/`.
- Do not loop identical failing commands (`tsc --noEmit` / vitest) across the
  whole repo when only one module is under review; narrow scope first.
- `import type { FooService }` cannot be instantiated in tests; use a value
  import when the constructor creates it under test.

## Troubleshooting

- Module scoped service tests: run vitest from the `apps/server-nestjs` working
  directory with `src/modules/<name>/<name>.spec.ts`
- For Prisma unique-input args, match generated types (often compound keys like
  `{ pluginName_key: { ... } }`)
- If `@Inject` is used in a service, import `Inject` from `@nestjs/common`
- If a vitest command fails with "No test files found" or identical repeat
  output, change strategy (path, working dir, or test scope) before rerunning.
  Repeating the exact same failing command is a blocker.
- Update `MODULARISATION-STATUT.md` after every migration: module section, route
  totals, key date, changelog.
- `tsc --noEmit` may report unrelated module failures; for a new module
  migration, limit blocking judgment to its own file unless the user asks for a
  whole-repo clean sweep.
