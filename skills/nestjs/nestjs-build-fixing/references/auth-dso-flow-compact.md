# DSO Token Auth Flow — Compact Pattern

Resolve admin permissions at auth time, not in the guard.

## Flow

```text
AdminGuard
  └→ AuthService.authenticateHeaders(request.headers, requirements)
       ├→ authenticateDsoToken()
       │     ├→ validateToken(rawToken, requirements)
       │     │      → TokenUserResult (raw)
       │     ├→ resolveAdminPermissions(tokenResult)
       │     │      → bigint (resolved)
       │     └→ returns UserContext { userId, adminPermissions, userType }
       │            ← no tokenResult/tokenKind
       └→ authenticateBearerToken()
             └→ keycloakJwtService.validatePayload(payload, reqs)
                    → UserContext already resolved
  └→ AdminService.validate(policy, user)  ← pure validation, no Prisma
```

## Rules

- **`resolveAdminPermissions` is private on `AuthService`** — not on
  `AdminService`, not on `AdminPolicy`. It queries `adminRole.findMany` for
  global roles and merges with token/user role permissions.
- **No `tokenResult` or `tokenKind` on `UserContext`** — permissions are
  resolved before `authenticateHeaders` returns. The guard never re-fetches the
  token.
- **`AdminService` is pure logic** — no constructor injection, no Prisma
  dependency. `validate()` only checks the already-resolved `adminPermissions`
  bitmask against the policy.
- **`AdminPolicy` owns only `build(context)`** — pure reflector operation. No
  Prisma, no service dependency.

## Pitfalls

- **Don't store the raw `TokenUserResult` on the request or `UserContext`** — it
  creates stale-data risk and couples the request shape to internal token
  resolution. Let `authenticateDsoToken` own the full lifecycle.
- **Keycloak bearer tokens already have resolved permissions** —
  `keycloakJwtService.validatePayload` returns `UserContext` with
  `adminPermissions` set. No need to call `resolveAdminPermissions` for bearer
  tokens.
- **When mocking in tests**, `prisma.adminRole.findMany` must be mocked (return
  `[]`) in `beforeEach` for any test that exercises the DSO path, since
  `resolveAdminPermissions` calls it.
