# Workspace package exports under Vite 8

When a workspace package is imported at runtime by another workspace package,
export the built artifacts from `dist/`, not the source files.

Pattern:

- `main`/`module`/`exports.import` -> `./dist/index.js`
- `exports.types` -> `./dist/index.d.ts`
- keep source entrypoints only for authoring, not runtime resolution

Why this matters:

- Vite 8/Rolldown and Node resolution are stricter than Vite 7/esbuild
- a package that still exports `src/index.ts` can fail to resolve during tests
  or SSR-like import paths even if the source compiles fine

Validation checklist:

1. Inspect each plugin/package `package.json`
2. Confirm runtime exports point at `dist/index.js`
3. Rebuild the package after changing exports
4. Re-run the server test that imports the package indirectly

Observed in this repo:

- `@cpn-console/*-plugin` packages needed export updates so `apps/server` could
  resolve them during `prepare-app.spec.ts`
- the fix was to stop pointing exports at source entrypoints and use built JS
  artifacts instead
