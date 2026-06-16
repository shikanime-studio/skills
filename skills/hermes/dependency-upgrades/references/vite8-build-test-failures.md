# Vite 8 Upgrade: Build/Test Failure Patterns

Captured from a pnpm monorepo upgrade where `pnpm build` and `pnpm test` failed
after Vite 8.

## CSS minification failure with DSFR

Symptom: Vite build fails during CSS minification with invalid legacy media
queries from DSFR, such as IE-only hacks like `@media (min-width: 0\0)`.

**Fix: add `css.lightningcss.errorRecovery: true` to `vite.config.ts`.** This is
the standard approach for third-party CSS containing legacy IE hacks that
Lightning CSS rejects. The hack is inert in modern browsers.

```ts
css: {
  lightningcss: {
    errorRecovery: true,
  },
}
```

DSFR 1.14.x (latest, 3 months old as of 2026-06) still ships these hacks with no
upstream fix. An alternative is using DSFR's `*.main.min.css` bundles (which
exclude legacy hacks) — but only if the project controls its own DSFR import
paths and the bundles cover all needed styles.

## zod-validation-error v3 exports field rejected by Vite 8 resolver

Symptom: Vitest tests fail with
`"./v3" is not exported under the conditions ["node", "development", "import"]
from package zod-validation-error`
even though the package's `exports` field maps `"."` to `"./v3/index.mjs"`.

Root cause: `zod-validation-error@3.x` uses non-standard export keys
(`"require"` and `"import"` at the top level of the exports map) instead of the
standard conditions-based format. Vite 8's Rolldown-based resolver is stricter
about exports field evaluation and rejects this format when resolving with
`["node", "development", "import"]` conditions.

Note: This is distinct from the v5 issue (where `/v3` is needed for Zod version
compatibility). Here the package is already v3-compatible; the issue is purely
the exports field format.

Preferred fix: remove the dependency if it is only used for lightweight
formatting. In `cloud-pi-native/console`, `zod-validation-error` was only used
by `parseZodError`, so the durable fix was:

1. Replace `fromZodError(zodError).toString()` with a local formatter over
   `ZodError.issues`.
2. Preserve the existing message shape expected by tests:
   `Validation error: <message> at "<path>"`, with `;` between issues and `, or`
   between union alternatives.
3. Remove `zod-validation-error` from `packages/shared/package.json` and all
   lockfile entries.
4. Remove the Vite pre-resolve plugin from `apps/server/vite.config.ts`.
5. Verify with the package that owns the formatter and a focused server Vitest
   smoke test:

```sh
pnpm --filter @cpn-console/shared lint
pnpm --filter @cpn-console/shared build
pnpm --filter @cpn-console/shared test
pnpm --filter @cpn-console/server exec vitest run <small-server-spec>
rg -n "zod-validation-error|zod-validation-error-resolver" .
```

Minimal local formatter shape:

```ts
import type { ZodError, ZodIssue } from "zod";

const issueSeparator = "; ";
const unionSeparator = ", or ";
const identifierRegex = /^[A-Z_$][\w$]*$/i;

function formatPath(path: ZodIssue["path"]): string {
  if (path.length === 1) return String(path[0]) || '""';
  return path.reduce<string>((acc, propertyKey) => {
    if (typeof propertyKey === "number") return `${acc}[${propertyKey}]`;
    const escapedKey = propertyKey.replace(/"/g, '\\"');
    if (!identifierRegex.test(propertyKey)) return `${acc}["${escapedKey}"]`;
    return `${acc}${acc.length === 0 ? "" : "."}${propertyKey}`;
  }, "");
}

function formatIssue(issue: ZodIssue): string {
  if (issue.code === "invalid_union") {
    return issue.unionErrors
      .map((error) => error.issues.map(formatIssue).join(issueSeparator))
      .filter((message, index, messages) => messages.indexOf(message) === index)
      .join(unionSeparator);
  }
  if (issue.path.length > 0) {
    if (issue.path.length === 1 && typeof issue.path[0] === "number")
      return `${issue.message} at index ${issue.path[0]}`;
    return `${issue.message} at "${formatPath(issue.path)}"`;
  }
  return issue.message;
}

export function parseZodError(zodError: ZodError) {
  if (zodError.issues.length === 0) return zodError.message;
  return `Validation error:
    ${zodError.issues.map(formatIssue).join(issueSeparator)}`;
}
```

Fallback: if removing the dependency would duplicate too much behavior, add a
pre-resolve Vite plugin that redirects the import to the concrete `.mjs` file,
bypassing exports map evaluation entirely. Treat this as a last resort because
it hardcodes package-manager layout and keeps the problematic dependency in the
graph.

Concrete plugin pattern (validated in `apps/server/vite.config.ts`):

```ts
import { resolve as resolvePath } from "node:path";

const zveEntry = resolvePath(
  fileURLToPath(
    new URL(
      "../../packages/shared/node_modules/zod-validation-error/v3/index.mjs",
      import.meta.url,
    ),
  ),
);

export default defineConfig({
  plugins: [
    {
      name: "zod-validation-error-resolver",
      enforce: "pre",
      resolveId(source) {
        if (
          source === "zod-validation-error" ||
          source === "zod-validation-error/v3"
        ) {
          return zveEntry;
        }
        return undefined;
      },
    },
  ],
  // ...
});
```

Key: `enforce: 'pre'` ensures this runs before Vite's built-in resolver. Return
the absolute path to the `.mjs` entry file for both the bare specifier and the
`/v3` subpath. After adding the plugin, rebuild workspace packages that compile
the import (`pnpm --filter <pkg> run build`) so downstream JS resolves correctly
at runtime.

## zod-validation-error v5 with Zod 3

Symptom: shared tests fail after dependency churn because `zod-validation-error`
v5 default exports target newer Zod behavior while the workspace still uses
Zod 3.

Fix: import the explicit v3 entry point.

```ts
import { fromZodError } from "zod-validation-error/v3";
```

This is the **documented and correct** API — v5 ships both v3 and v4 compat
layers. The `/v3` subpath is not a workaround. Keep the existing Zod 3 types
unless intentionally migrating Zod itself.

## Gitbeaker 43 API shape change

Symptom: TypeScript build fails on `GroupMembers.add` calls after Gitbeaker is
upgraded.

Old shape:

```ts
client.GroupMembers.add(group.id, userId, accessLevel);
```

New shape:

```ts
client.GroupMembers.add(group.id, accessLevel, { userId });
```

Apply to both plugin implementations and NestJS rewrite services. Update
unit-test mocks/fixtures to match the new argument shape and Gitbeaker types.

## Vitest V8 coverage parses non-code assets

Symptom: `vitest run --coverage` passes tests, then V8 coverage reports Rolldown
parse errors for files such as
`apps/server/src/prisma/migrations/*/migration.sql`:

```text
Failed to parse .../migration.sql. Excluding it from coverage.
RolldownError: Parse failed with 1 error:
Expected a semicolon or an implicit semicolon after a statement
1: -- AlterTable
2: ALTER TABLE "Project" ADD COLUMN "slug" TEXT;
```

Root cause: coverage `include: ['src/**']` collects every file under `src`,
including Prisma SQL migrations, `.prisma` schemas, and `.toml` assets. With
Vitest 4/V8 coverage, Rolldown attempts to parse uncovered files during coverage
remapping.

Fix: restrict coverage includes to code extensions instead of a directory-wide
glob.

```ts
coverage: {
  provider: 'v8',
  include: ['src/**/*.ts'],
  exclude: [
    '**/*.spec.ts',
    '**/*.d.ts',
    // existing excludes
  ],
}
```

Verification: run a focused passing spec with coverage so the command reaches
the coverage generation phase even if the full suite has unrelated failures.

```sh
pnpm --filter @cpn-console/server exec vitest run --coverage
  src/utils/date.spec.ts
```

A clean coverage report without `.sql` parse messages validates the glob fix. If
the full suite still fails, separate those failures from the coverage remapping
issue.

## Prisma generate race in parallel tests

Symptom: `pnpm -r --no-bail run test` fails intermittently with
`EACCES: permission denied, copyfile` on `libquery_engine.node` when two
workspaces run `prisma generate` concurrently.

Fix: serialize root test scripts with `--workspace-concurrency=1` in root
`package.json`:

```json
{
  "test": "pnpm -r --workspace-concurrency=1 --no-bail run test"
}
```

## Build serialization: packages building before their dependencies finish

Symptom: `pnpm -r run build` fails with
`TS2307: Cannot find module '@cpn-console/shared'` in packages like `test-utils`
that depend on `shared`, even though `shared` is declared as a `workspace:^`
dependency.

Root cause: `pnpm -r run build` runs builds in parallel across the workspace.
Even though pnpm respects dependency order for _starting_ builds, a large
workspace build (`shared` produces `types/` output via `tsc`) can still be
running when a dependent package (`test-utils`) starts its own `tsc` that needs
those types.

Fix: add `--workspace-concurrency=1` to the root `build` script:

```json
{
  "build": "pnpm -r --workspace-concurrency=1 run build"
}
```

This ensures `packages/shared` fully completes (including `types/` emission)
before `packages/test-utils` starts. This is a build orchestration issue, not a
code issue.

## Docker build: workspace package types not resolvable

Symptom: Docker build fails at `vue-tsc --noEmit` (or `tsc`) with
`TS2307: Cannot find module '@cpn-console/shared'` for every import, even though
the shared package is built in an earlier Dockerfile layer.

Root cause: `pnpm install` runs before all workspace sources are copied into the
Docker context. When workspace sources are copied in multiple stages (packages
first, apps later), the initial `pnpm install` doesn't create proper workspace
symlinks for packages copied in later stages. This affects all workspace
packages copied after the initial install, not just the client.

Fix: add a second `pnpm install --ignore-scripts` after copying all workspace
sources:

```dockerfile
# After ALL workspace sources are in place:
COPY --chown=node:root apps/client/ ./apps/client/
RUN pnpm install --ignore-scripts
RUN pnpm --filter client run icons
```

This re-syncs all workspace links so `vue-tsc` with `Bundler` resolution (or
`tsc` with `NodeNext`) can find every workspace package's built types/output.

## `patch` tool brace matching in nested config

Symptom: After using `patch` mode=replace to insert into a nested config object
(e.g., `defineConfig({...})`), the resulting file has duplicated `},` or `}`
lines at the insertion point, causing syntax errors.

Root cause: The patch tool can mis-count closing braces when the `old_string`
ends at a brace boundary.

Fix: Always verify the resulting file with `read_file` after patching. Look for
duplicated `}` or `},` lines at the insertion point. If the patch created a
syntax error, re-patch with adjusted `old_string` that includes unique
surrounding context (e.g., include the line _after_ the insertion point in
`old_string`).

## stylelint `no-invalid-position-declaration` false positive on Vue inline styles

Symptom: After lockfile churn from a Vite upgrade, `pnpm run lint` fails with
`no-invalid-position-declaration` errors on Vue components that use inline
`style` attributes (e.g., `style="white-space: pre-wrap;"`,
`style="display: none;"`, `style="width: fit-content"`).

Root cause: stylelint 17.x added a built-in `no-invalid-position-declaration`
rule that flags CSS declarations not inside a rule block. Vue template inline
`style` attributes are parsed as standalone declarations, triggering the rule as
a false positive — they are valid HTML.

**Preferred fix: convert inline styles to utility classes or scoped CSS** (user
explicitly rejected disabling the rule):

| Inline style                     | Replacement                                     |
| -------------------------------- | ----------------------------------------------- |
| `style="white-space: pre-wrap;"` | `class="whitespace-pre-wrap"`                   |
| `style="display: none;"`         | `class="hidden"`                                |
| `style="width: fit-content"`     | `class="w-fit"`                                 |
| `style="field-sizing: content"`  | Scoped CSS class (no UnoCSS utility equivalent) |

For `field-sizing: content`, add a scoped class:

```vue
<template>
  <DsfrInputGroup class="field-sizing-content" />
</template>

<style scoped>
.field-sizing-content {
  field-sizing: content;
}
</style>
```

Fallback (only if converting is impractical): disable the rule in
`.stylelintrc.js`:

```js
rules: {
  'no-invalid-position-declaration': null,
}
```

The `--fix` flag cannot auto-fix this because there is no reordering to apply —
the declarations are valid, just in a position the rule doesn't understand for
Vue templates.

## Verification sequence

Run the exact failing commands after each focused fix:

```sh
pnpm build
pnpm test
```

Do not stop at type-check success; Vite CSS minification and Vitest dependency
resolution can fail later in the pipeline.
