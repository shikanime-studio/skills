# Vitest coverage fix: Prisma migrations in pnpm monorepo

## Context

- Repo: `cloud-pi-native/console`
- App: `apps/server` (Fastify 4, Prisma 6, Vitest 4.1.5)
- Monorepo: pnpm v10.33 with workspace packages
- PR: 2191 "Add support for Keycloak based token"

## Failure

CI job `unit-tests / Unit tests` failed with:

```
Error [RollupError]: Expected ';', '}' or <eof>
...
Excluding it from coverage.
Failed to parse file:///.../prisma/migrations/20250916134454_add_project_resources/migration.sql
...
FAIL  src/prepare-app.spec.ts [ src/prepare-app.spec.ts ]
```

V8 coverage tried to parse `.sql` migration files as JavaScript.

## Resolution

Added `'**/*.sql'` to the coverage `exclude` list in `apps/server/vitest.config.ts`.

```ts
coverage: {
  provider: 'v8',
  exclude: [
    '**/types',
    '**/mocks',
    '**/*.spec.ts',
    '**/*.d.ts',
    '**/*.vue',
    '**/queries.ts',
    '**/mocks.ts',
    '**/*.sql',
  ],
}
```

## Why this pattern

- `apps/server` test root is in `apps/server/`
- Prisma migrations live at `apps/server/src/prisma/migrations/**/*.sql`
- `vitest --coverage` on `apps/server` resolves those as relative files
- `'**/*.sql'` catches them regardless of nesting depth

## Notes

- The lockfile regenerated with workspace-specific tags (`file:plugins/argocd(...)`). Do not commit lockfile changes unless intentionally updating dependencies.
- `prepare-app.spec.ts` imports the app and DB connections; coverage of migration SQL is never needed.
