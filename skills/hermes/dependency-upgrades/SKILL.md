---
name: dependency-upgrades
description:
  "Upgrade dependencies in a pnpm monorepo — surgical targeted upgrades vs
  blanket major bumps, Vite 8 migration patterns, and avoiding cascading
  breaking changes. Triggered when user asks to upgrade a specific dependency,
  migrate to a new major version, or mentions taze/upgrade/update packages."
---

# Dependency Upgrades in a Monorepo

Surgical dependency upgrades in a pnpm workspace — avoid the cascade of breaking
changes that blanket tools introduce.

## Core Principle

**Never use `taze major -r -w` for a targeted upgrade.** It upgrades every
package across every workspace package, introducing dozens of unrelated breaking
changes that must be manually reverted. The user asked for Vite — give them
Vite, not a new TypeScript major.

## Strategy: Surgical Upgrade

### 1. Identify the scope

Before touching anything, read the migration guide for the target package.
Identify:

- Which packages in the workspace depend on it
- What config changes are required
- What peer dependency constraints exist

### 2. Update only the target package

Use `patch` to update version strings in each `package.json` — one call per
package, not a blanket tool:

```text
# For each package that depends on the target:
patch mode=replace path=<package.json> old_string='"vite": "^7.x.x"'
  new_string='"vite": "^8.x.x"'
```

If using taze, scope it with `--include`:

```text
npx taze --include vite --write
```

Verify with `search_files` that only the intended packages changed.

### 3. Check for config changes

Read the upstream migration guide (usually `https://<pkg>.dev/guide/migration`).
Key things to check:

- Deprecated config options that were removed
- New required config options
- Changed defaults that affect behavior

### 4. Install and verify

```text
pnpm install
pnpm build  # or the relevant build command
pnpm test   # run the test suite
```

### 5. Fix issues one at a time

If tests fail, read the actual error. Don't guess. Each error tells you exactly
what changed.

## Vite 7 to Vite 8 Specifics

Vite 8 replaces esbuild/Rolldown with Rolldown/Oxc. Key config changes:

### build.target

`build.target: 'ESNext'` (uppercase) is **not valid** for Lightning CSS (Vite
8's default CSS minifier) — it throws `Unsupported target "ESNext"`. The fix is
lowercase `'esnext'`. Other valid values:

- `'baseline-widely-available'` (Vite 8 default, recommended for broader browser
  support)
- A specific browser target like `'chrome111'`

### CSS Minification

Lightning CSS is stricter than esbuild's CSS minifier. If the project uses CSS
with legacy hacks (e.g., `@media (min-width: 0\0)` from DSFR), add
`css.lightningcss.errorRecovery: true` to `vite.config.ts`:

```ts
css: {
  lightningcss: {
    errorRecovery: true,
  },
}
```

This is the standard fix for third-party CSS that contains legacy IE hacks. The
hack is inert in modern browsers and stripping it is safe. DSFR 1.14.x (latest)
still ships these hacks in its compiled CSS with no upstream fix available.

An alternative is to use DSFR's `*.main.min.css` bundles (which exclude legacy
hacks) instead of the all-in-one `dsfr.min.css` import — but only if the project
controls its own DSFR import paths and the bundles provide all needed styles.

### esbuild config options

The `esbuild` config option is deprecated in favor of `oxc`. For dependency
optimization, `optimizeDeps.esbuildOptions` is deprecated in favor of
`optimizeDeps.rolldownOptions`. Vite auto-converts these, but migrate away from
them.

### No native decorator lowering

Oxc does not support lowering native decorators. If the project uses TypeScript
decorators (e.g., NestJS), add a Babel or SWC plugin workaround.

## Pitfalls

- **Post-upgrade failures can come from adjacent dependency churn**: after a
  Vite 8 upgrade, `pnpm build`/`pnpm test` failures may be from packages
  upgraded in the same lockfile churn, not Vite itself. Known fixes: if
  `zod-validation-error` was removed due to Vite 8 incompatibility, use Zod's
  native error formatting instead — `z.treeifyError()`, `z.prettifyError()`, and
  `z.flattenError()` are built into zod@3.x and provide the same functionality
  without the non-standard `exports` field that Vite 8 rejects. Update Gitbeaker
  43 `GroupMembers.add` calls to
  `GroupMembers.add(groupId, accessLevel, { userId })`, and restrict Vitest
  coverage globs to code file extensions when V8/Rolldown tries to parse assets
  like Prisma SQL migrations. See `references/vite8-build-test-failures.md`.
- **`taze major -r -w` upgrades everything**: Zod 3->4, TypeScript 5->6, pinia
  2->3, vue-router 4->5 — all in one shot. Each is a major breaking change
  requiring code modifications. **This is the #1 cause of cascading breakage in
  monorepo upgrades.** Always scope taze with `--include` or use `patch` for
  surgical updates. If `taze major` has already run, search every `package.json`
  in the workspace for newly-introduced `^6.x`, `^4.x`, `^5.x`, `^3.x` versions
  and revert non-target packages before `pnpm install`.
- **pnpm trust policy / lockfile installs in pnpm 11**: Fresh installs can hit
  `ERR_PNPM_TRUST_DOWNGRADE` because pnpm 11 supply-chain verification is
  stricter than earlier versions. Two cheap bypasses:
  1. `pnpm install --trust-policy=low` — verified working in this repo when
     trust checks blocked install.
  2. `--trust-lockfile` appears in `pnpm install --help` but is rejected at
     runtime in pnpm 11; do not rely on it. Note:
     `pnpm config set trustPolicyExclude ...` didn’t take effect in this repo
     despite being stored; prefer the explicit install flag over config
     mutations when a one-off install is needed.
- **pnpm patches**: If a patched dependency gets a new version (even
  patch-level), the old hash won't match and installation fails at the end with
  `ERR_PNPM_PATCH_FAILED`. Example: `@gouvminint/vue-dsfr` pinned to `8.15.0`
  with `^8.15.0` will resolve to `8.17.0` due to `minimumReleaseAge`. Fix:
  either pin the exact version (`"8.15.0"` without `^`), remove the patch from
  `pnpm-workspace.yaml` temporarily, or re-create the patch for the new version.
- **`minimumReleaseAge` in pnpm-workspace.yaml**: With
  `minimumReleaseAge: 1440`, pnpm will resolve to the latest version matching
  the semver range. Always pin exact versions for patched dependencies.
- **patched dependency drift + `minimumReleaseAge`**: When `patchedDependencies`
  references a version that has moved past `minimumReleaseAge`, installs can
  still run but end with `ERR_PNPM_PATCH_FAILED` because the resolved tarball
  hash no longer matches the patch metadata. Immediate check path: grep
  `patches/` and `patchedDependencies` in `pnpm-workspace.yaml`, then match
  against actual resolved versions in `pnpm-lock.yaml`. Remediation order: 1)
  pin exact dependency versions in `package.json` if the range is `^x.y.z`, 2)
  remove the affected patch entry from `patchedDependencies` temporarily and use
  the unpatched upstream build until a refreshed patch is created, 3) if the
  patch entry is stale but unneeded, clean with `pnpm clean --lockfile` followed
  by `pnpm install`; this rebuilds resolution from the same constraints without
  regenerating topological lockfile structure. `pnpm install --force` will not
  bypass a patch-hash mismatch, because patches are applied after download and
  the hash still must match.

### Vite 8 stricter exports resolution

Vite 8's Rolldown-based resolver is stricter about `package.json` `exports`
fields than Vite 7's esbuild-based resolver. Packages that use non-standard
export keys (e.g. `"require"` and `"import"` at the top level instead of nested
under `"conditions"`) may fail with
`"./subpath" is not exported under the conditions [...]` errors. This affects
packages like `zod-validation-error@3.x`.

Preferred fix order:

1. **Use built-in alternatives first.** Zod v3 already provides
   `z.treeifyError()`, `z.prettifyError()`, and `z.flattenError()` — these
   replace `zod-validation-error` entirely. Remove the dependency and use the
   native API.
2. If the dependency is only used for a small helper with no built-in
   equivalent, remove it and implement the helper locally.
3. If removal would duplicate too much behavior, upgrade only the problematic
   package after checking peer dependencies and subpath compatibility.
4. Use a Vite pre-resolve plugin to redirect to a concrete entry file only as a
   last resort, because it hardcodes package-manager layout and keeps the broken
   export map in the dependency graph.

After adding a Vite pre-resolve plugin, **rebuild any workspace packages that
compile the import** (e.g. `pnpm --filter @cpn-console/shared run build`) so
downstream compiled JS picks up the correct resolution path. Compiled `.js`
output still carries the original import specifier — resolved at runtime by the
plugin, not at compile time.

### Repo-specific warning cleanup

When the Vite build is only noisy because a third-party CSS bundle ships legacy
IE hacks, prefer the simplest quiet-build fix:

1. First verify whether the warning is coming from vendored CSS or from code we
   own.
2. If the warning is from DSFR or similar third-party CSS and the team wants a
   quiet build, switch CSS minification to `esbuild` rather than carrying a
   permanent Lightning CSS recovery shim.
3. If the warning references a missing CSS asset, add the asset under
   `apps/client/public/` and make the path exact.
4. If the warning is `INEFFECTIVE_DYNAMIC_IMPORT` on a barrel that is already
   needed eagerly, replace repeated dynamic imports with a single static import
   list.

This repo used that pattern successfully for:

- DSFR Lightning CSS warnings
- `/icons/government-fill-blue.svg` runtime resolution
- `packages/shared/dist/contracts/index.js` dynamic-import noise

### Workspace package exports and build ordering

When a workspace package is imported at runtime by another workspace package,
its `package.json` exports should point to `dist/index.js` and
`dist/index.d.ts`, not to source entrypoints like `src/index.ts`. Vite
8/Rolldown and Node resolution are stricter than Vite 7, so source-path exports
can fail during indirect imports even when the package builds successfully.

If a tool needs manifest metadata, add `"./package.json": "./package.json"` as a
compatibility subpath. This does not change runtime code loading; it only allows
metadata lookups through the package resolver.

Build ordering guidance:

- `pnpm -r run build` can race when one workspace needs another workspace's
  emitted `dist/` or `types/`
- `--workspace-concurrency=1` is a safe orchestration fallback
- the better long-term fix is to make the dependency graph explicit with
  TypeScript project references and graph-aware builds (`tsc -b` or equivalent)

After changing exports, rebuild the affected package before re-running
downstream tests.

### Shared-package refactors: keep declaration emit in mind

When splitting shared contract wiring into a new module, verify that `tsc` still
emits clean declarations before keeping the split.

Pitfalls:

- Exporting a raw generic instance from `@ts-rest/core` can trigger TS4023 in
  declaration emit.
- If that happens, prefer a tiny wrapper helper that constructs the instance
  internally and returns the contract shape, rather than exporting the raw
  instance type.
- If the wrapper still leaks unnamed types, revert the split and keep the
  contract construction colocated with the consumer until a safer typing
  boundary is found.

Verify with both `pnpm build` and `pnpm test:cov` before treating the refactor
as complete.

See `references/workspace-plugin-exports.md` and
`references/vite8-workspace-exports.md` for the validation checklist and the
repo-specific pattern. See `references/pnpm11-trust-policy-bypass.md` for the
pnpm 11 install bypass pattern.

### Post-upgrade lint cleanup: stylelint `no-invalid-position-declaration`

After upgrading Vite (or any dep that triggers `pnpm install` with lockfile
churn), `pnpm run lint` may surface new stylelint 17.x rule
`no-invalid-position-declaration`. This rule flags CSS declarations outside of
rule blocks — a **false positive for Vue template inline `style` attributes**
(e.g., `style="white-space: pre-wrap;"`).

**Preferred fix: convert inline styles to utility classes or scoped CSS.** The
user explicitly rejected disabling the rule — the violations should be resolved
for real:

| Inline style                     | Replacement                              |
| -------------------------------- | ---------------------------------------- |
| `style="white-space: pre-wrap;"` | `class="whitespace-pre-wrap"`            |
| `style="display: none;"`         | `class="hidden"`                         |
| `style="width: fit-content"`     | `class="w-fit"`                          |
| `style="field-sizing: content"`  | Scoped CSS class (no utility equivalent) |

For properties without a utility equivalent (e.g., `field-sizing`), add a scoped
CSS class in the component's `<style scoped>` block.

Fallback (user rejected): disable the rule in `.stylelintrc.js` with
`'no-invalid-position-declaration': null`. This is only acceptable if converting
inline styles is impractical.

See `references/vite8-build-test-failures.md` for the local formatter pattern
and fallback resolver pattern.

### Generated `.d.ts` files are overwritten by `tsc`

The `types/*.d.ts` files emitted by `tsc` are **regenerated on every build**
from the `.ts` source files. Editing a generated `types/index.d.ts` directly is
futile — the next `tsc run` overwrites it.

**The fix goes in the source `.ts` file** (e.g. `src/index.ts`), not the
generated `.d.ts`.

Example: to make `VaultProjectApi` importable from `@cpn-console/vault-plugin`
(root), add the re-export in `src/index.ts`:

```ts
// src/index.ts
import { VaultProjectApi } from "./vault-project-api.js";
import { VaultZoneApi } from "./vault-zone-api.js";
export { VaultProjectApi, VaultZoneApi };
```

Then update consumers from deep paths to root:

```ts
// Before:
import type { VaultProjectApi } from "@cpn-console/vault-plugin/types/vault-project-api.js";
// After:
import type { VaultProjectApi } from "@cpn-console/vault-plugin";
```

Triple-slash references in `env.d.ts` files should also use root:

```ts
// Before:
/// <reference types="@cpn-console/vault-plugin/types/index.d.ts" />
// After:
/// <reference types="@cpn-console/vault-plugin" />
```

Simplify `package.json` `exports` to root-only once all deep-path imports are
removed:

```json
"exports": {
  ".": {
    "types": "./types/index.d.ts",
    "import": "./dist/index.js",
    "default": "./dist/index.js"
  }
}
```

### Monorepo build orchestration

Prefer a graph-aware recursive build before falling back to full serialization.

- `pnpm -r --sort run build` topologically orders workspace builds, which is
  usually enough when packages depend on sibling build outputs.
- Use `--workspace-concurrency=1` only when the graph is still implicit or a
  specific package has side effects that can race.
- If a package’s `prebuild` runs `pnpm i`, make that step idempotent or move it
  out of the recursive build path; otherwise recursive builds can contend on the
  same install/workspace state.

Example fallback:

```json
{
  "build": "pnpm -r --sort run build"
}
```

If sorting is not enough, temporarily serialize:

```json
{
  "build": "pnpm -r --workspace-concurrency=1 run build"
}
```

### Monorepo test concurrency

When two or more workspaces run `prisma generate` (or any binary-copy step) in
parallel via `pnpm -r --no-bail run test`, they can race on the same output file
and one fails with `EACCES: permission denied, copyfile`. In those cases,
serialize root test scripts with `--workspace-concurrency=1`:

```json
{
  "test": "pnpm -r --workspace-concurrency=1 --no-bail run test",
  "test:cov": "pnpm -r --workspace-concurrency=1 --no-bail run test:cov"
}
```

This is a build/test orchestration issue, not a code issue — the tests
themselves are fine when run serially.

### Docker builds: re-run pnpm install after copying all sources

When a Dockerfile copies workspace sources in multiple stages (packages first,
apps later), `pnpm install` run before all sources are copied won't create
proper workspace symlinks for packages copied later. This causes `vue-tsc` with
`Bundler` resolution (or `tsc` with `NodeNext`) to fail with
`TS2307: Cannot find module '@workspace/pkg'`. This affects **all** workspace
packages copied after the initial install, not just the client.

Fix: add `RUN pnpm install --ignore-scripts` after the last `COPY` of workspace
sources, before running any build/type-check commands.

### `patch` tool brace matching in nested config

When inserting into a nested config object (e.g., `defineConfig({...})`), the
`patch` tool can mis-count closing braces if `old_string` ends at a brace
boundary. This duplicates `},` lines and creates syntax errors. Always verify
with `read_file` after patching — look for duplicated `}` or `},` lines at the
insertion point. If the patch created a syntax error, re-patch with `old_string`
that includes unique context _after_ the insertion point.

### `execute_code` vs `terminal`

The `terminal` tool's heuristics frequently misclassify `vite build`, `tsc`, and
other one-shot build commands as "long-lived server/watch processes" and refuse
to run them (error: _This foreground command appears to start a long-lived
server/watch process_). Use `execute_code` with `subprocess.run()` for all
build, test, and type-check commands instead. Reserve `terminal` for truly
long-running processes (dev servers, watchers) and quick diagnostic commands.

### Audit scope after taze

After any `taze`-driven upgrade, run `search_files` for the target package name
across **all** workspace `package.json` files. `taze major -r -w` touches every
workspace package — not just the ones with the target dependency. Common
victims: `typescript`, `@types/node`, `zod`, `vitest`, `@vitest/coverage-v8`,
`pinia`, `vue-router`, `short-uuid`, `vitest-mock-extended`. Revert any
non-target package before installing.

## Reference

See `references/vite7-to-vite8.md` for the full Vite 7->8 migration checklist
and config mapping tables. See `references/vite8-build-test-failures.md` for
specific build/test failure patterns and fixes. See
`references/vite8-warning-cleanup.md` for the repo-specific warning cleanup that
removed LightningCSS and ineffective-dynamic-import noise. See
`references/plugin-type-reexports.md` for the pattern of consolidating plugin
type imports to root package (source `.ts` re-exports, not generated `.d.ts`
edits). See `references/monorepo-build-graph.md` for the topological build /
install-guard pattern that replaced blunt workspace serialization here. See
`references/ts-rest-contract-instance-ts4023.md` for the shared-contract
declaration-emit pitfall and the safer wrapper fallback.
