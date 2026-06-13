# Migration steps (session note)

1. Inspect legacy route source in `apps/server/src/resources/<module>/` and
   shared contract/schema in `packages/shared/src/contracts/<domain>.ts` and
   `packages/shared/src/schemas/<domain>.ts`.
2. Create new `apps/server-nestjs/src/modules/<name>/` files: controller,
   service, module, spec.
3. Use `AdminGuard` + `@RequireAdminPermission(...)` for admin routes, or
   project guards/decorators for project-scoped routes.
4. Validate request bodies with `ZodValidationPipe` tied to shared schemas.
5. Wire the new module in `apps/server-nestjs/src/main.module.ts`.
6. Update
   `apps/server-nestjs/documentation/Modularisation-de-console-server/MODULARISATION-STATUT.md`:
   - add a module section with routes, permissions, files, and legacy
     differences
   - update route totals and percentage
   - add a key date entry
   - append a changelog entry
7. If user requested kanban tracking, preserve module metadata (name, route
   count, permissions, files) in the task item.
8. Run:
   - `pnpm --filter server-nestjs exec tsc --noEmit`
   - or module-scoped vitest from `apps/server-nestjs`:
     `pnpm --filter server-nestjs exec vitest run
     src/modules/<name>/<name>.spec.ts`
