# Vite 8 workspace exports and build ordering

When a workspace package is imported at runtime by another workspace package,
export built artifacts from `dist/`, not source files.

Pattern:

- `main` / `module` / `exports.import` -> `./dist/index.js`
- `exports.types` -> `./types/index.d.ts`
- `./package.json` -> `./package.json` only when tooling needs to read package
  metadata through package resolution

Why this matters:

- Vite 8 / Rolldown and Node resolution are stricter than Vite 7 / esbuild
- runtime imports by package name need actual emitted JS, not TypeScript source
  paths
- `./package.json` is a compatibility escape hatch for metadata lookups, not a
  runtime code path

Use `./package.json` when:

- the package is inspected by tooling that resolves `package.json` as a subpath
- a loader or manifest reader expects to fetch package metadata through the
  module resolver

Do not use it as a workaround for broken runtime resolution. If the package
itself does not resolve, fix the `dist/` exports first.

Build ordering guidance:

- `pnpm -r run build` can race when one workspace needs another workspace's
  emitted `dist/` or `types/`
- `--workspace-concurrency=1` is a safe orchestration fallback
- the better long-term fix is to make the dependency graph explicit with
  TypeScript project references and graph-aware builds (`tsc -b` or equivalent)

Validation checklist:

1. Confirm runtime exports point at `dist/index.js`
2. Confirm `exports.types` points at generated declaration output
3. Rebuild the package after changing exports
4. Re-run the downstream consumer test or build that imports it indirectly
5. Prefer graph-aware builds over serializing the whole workspace when the repo
   grows
