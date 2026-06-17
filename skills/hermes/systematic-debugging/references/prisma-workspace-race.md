# Prisma workspace generation race

Observed in `console.project-migration`:

- `pnpm -r --no-bail run test` could fail intermittently when `apps/server` and
  `apps/server-nestjs` ran in parallel.
- The failure appeared during `pretest -> prisma generate` with errors such as:
  - `EACCES: permission denied, copyfile ... libquery_engine.node -> .../.prisma/client/libquery_engine.node`
  - transient `SyntaxError: Invalid or unexpected token` from the generated
    Prisma client entrypoint.

Root cause

- Both server packages generate into the same workspace-level Prisma client path
  under `node_modules/.pnpm/.../.prisma/client`.
- Parallel package execution races on that shared output.

Fix pattern

1. Serialize workspace test execution with
   `pnpm -r --workspace-concurrency=1 --no-bail run test`.
2. Re-run the full suite to confirm both `apps/server` and `apps/server-nestjs`
   generate cleanly in sequence.

Notes

- This is a runner/workspace orchestration issue, not a code-level bug in the
  app packages.
- Keep the change at the root script level so all package test entrypoints
  inherit the safer order.
