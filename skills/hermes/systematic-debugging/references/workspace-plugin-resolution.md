# Workspace plugin resolution (console repo, Vite 8 migration)

Session pattern:

- CI/Docker failed on `apps/server/src/prepare-app.spec.ts` and `apps/server`
  build.
- Error: `Failed to resolve entry for package "@cpn-console/argocd-plugin"` /
  TS2307 in `apps/server/src/plugins.ts`.

Root cause:

- `apps/server/src/plugins.ts` imported plugin packages eagerly at module load.
- The build container/test runner could not resolve the package entry during
  module evaluation.
- The real issue was architectural: runtime loading depended on package
  resolution too early.

Durable fix:

1. Declare each built-in plugin as an explicit `workspace:^` dependency of
   `apps/server`.
2. Load built-in plugins lazily with `await import(packageName)` from a fixed
   package-name list.
3. Await plugin-manager initialization before `prepareApp()` proceeds.
4. Keep package exports minimal: runtime entry on `dist/index.js`, types on
   `types/index.d.ts`.

Verification commands used:

- `pnpm --filter @cpn-console/server test:cov`
- `docker build -f apps/server/Dockerfile --target build .`

Pitfalls:

- Do not fix this with install hooks or workspace-concurrency flags.
- Do not reintroduce static package imports in `src/plugins.ts`.
- If a plugin package is missing from Docker/CI resolution, check
  `apps/server/package.json` dependencies first.
