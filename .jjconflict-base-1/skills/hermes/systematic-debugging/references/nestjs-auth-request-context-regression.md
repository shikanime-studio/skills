# NestJS auth request-context regression pattern

Session pattern:
- `UserGuard` authenticates and stores the payload on `request.user`.
- `UserContext` is the shared shape for authenticated requests.
- Downstream guards must read `request.user?.userId` and `request.user?.adminPermissions`.
- If a helper only builds headers, seed `request.user` directly in the spec body.

Common failures seen during the refactor:
- TS2459: a module imports `UserContext` from `./user.guard` but the guard no longer re-exports it.
- TS2339 / TS2551: code still reads `request.userId` or `request.adminPermissions` after the runtime shape moved under `request.user`.
- TS2345: controller/service calls pass `user.userId` while the shared interface still marks it optional.

Fix pattern:
1. Make the runtime shape canonical: `request.user = { userId, adminPermissions }`.
2. Re-export `UserContext` from `user.guard.ts` if downstream files import it there.
3. Tighten `UserContext` to required fields once authentication always returns them.
4. Re-run the auth guard specs plus the project service/controller specs that consume the shared type.
