# Auth request shape split

Session note for NestJS auth refactors and rebase conflict resolution.

## Final contract

- `UserGuard` authenticates and writes `request.user = { userId }` only.
- `UserContext` stays identity-only (`userId`), not permission-bearing.
- Permission data lives on the request/context side used by permission guards,
  not inside `request.user`.
- `AdminPermissionGuard` and similar guards should read
  `request.adminPermissions` with a `0n` fallback when absent.
- `ProjectPermissionGuard` should also read `request.adminPermissions`
  separately if it needs admin-level overrides.

## Rebase resolution pattern

1. Resolve the type shape first (`UserContext` vs request extensions).
2. Update the guard implementation and every spec/helper that constructs request
   mocks in the same pass.
3. Re-run focused auth specs, then the workspace suite.
4. Scan for conflict markers before trusting the tree.

## Practical checks

- Search for `adminPermissions` in
  `apps/server-nestjs/src/modules/infrastructure/auth`.
- Ensure `user.guard.spec.ts`, `user-type.guard.spec.ts`,
  `admin-permission.guard.spec.ts`, and request factories all match the same
  contract.
- Verify with `pnpm --filter @cpn-console/server-nestjs exec vitest run ...` and
  `pnpm run test`.
