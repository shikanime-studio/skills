---
name: vitest-configuration
title: Vitest Configuration & Coverage Debugging
description: Troubleshoot and fix Vitest configuration issues, with emphasis on coverage provider failures caused by non-JavaScript files being picked up by the transformer.
triggers:
  - vitest coverage failure
  - vitest config
  - coverage PARSE_ERROR
  - RollupError in vitest
  - test coverage excluding files
  - vitest --coverage fails
  - prisma migrations coverage
---

# Vitest Configuration & Coverage Debugging

## Problem: Coverage crashes on non-JS files

Vitest's V8 coverage provider remaps coverage using Rollup. If Rollup encounters a file it cannot parse as JavaScript/TypeScript (e.g. `.sql` migration files, raw `.json` data, `.css`), it throws `PARSE_ERROR` and the coverage step fails even if all tests pass.

### Symptoms

- `vitest run --coverage` fails with `Error [RollupError]: Expected ';', '}' or <eof>`
- Error path points to `.sql`, `.css`, `.json`, or other non-TS files under `src/`
- Test suite itself shows `363 passed | 2 skipped` but job exits non-zero
- CI reports `unit-tests / Unit tests` failure while lint/build pass

### Fix

Add the offending file pattern to the coverage `exclude` list in `vitest.config.ts`.

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
    '**/*.sql',          // <-- add this for Prisma migrations
    '**/*.css',          // <-- add this for imported styles
    '**/*.json',         // <-- add this for fixture data
  ],
}
```

Use glob patterns that match the actual offending paths. Common culprits:
- `**/prisma/migrations/**/*.sql` (generated migrations)
- Static assets imported by test files
- Fixture data files

## General Vitest Config Checklist

- `include` / `exclude` use glob patterns relative to `root`
- Coverage `include` should stay narrow (e.g. `src/**`) to avoid scanning `node_modules`
- Coverage `exclude` should list spec files, types, mocks, and any non-TS asset
- If using `pool: 'forks'`, ensure the environment has enough resources; `ENOSPC` during coverage may indicate temp-disk pressure, not a config bug

## References

- `references/vitest-coverage-sql-fix.md` — specific case: Prisma migrations breaking coverage in pnpm monorepo with `apps/server` root
