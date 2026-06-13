# Guard/Service Separation Pattern in NestJS

## Principle

Guards should be thin — like controllers, they delegate all domain logic to services. All authentication logic (token parsing, JWT verification, DB lookups) belongs in services. This avoids hidden dependency chains that cause `UnknownDependenciesException` when guards are used across module boundaries.

## Architecture

```
UserService (authentication domain logic)
  └── authenticate(request)
        ├── DSO token (x-dso-token header) → AuthService.validateToken()
        └── Bearer JWT (Authorization header) → JwtService.verifyAsync → KeycloakJwtService.validatePayload
  └── Returns UserContext { userId, adminPermissions }

UserGuard (thin guard — authentication only)
  └── Calls userService.authenticate(request)
  └── Sets request.userId, request.adminPermissions
  └── Only depends on UserService

AdminPermissionGuard (thin guard — authorization only)
  └── Calls userService.authenticate(request)
  └── Checks @RequireAdminPermission() metadata via Reflector
  └── Only depends on UserService + Reflector
```

## Why a Separate UserService?

Rather than putting `authenticateRequest()` on `AuthService`, create a dedicated `UserService`:

- `AuthService` handles token CRUD and token-based validation (`validateToken(hash)`)
- `UserService` handles the HTTP-level authentication flow (extract token from headers, dispatch to correct validation strategy)
- This follows single-responsibility: `AuthService` knows about tokens, `UserService` knows about HTTP requests

## Why This Matters

NestJS resolves guard dependencies in the **consuming** module's scope, not the **providing** module's scope. When `ProjectModule` uses `AdminPermissionGuard` via `@UseGuards()`, NestJS tries to instantiate it within `ProjectModule`'s scope. If the guard depends on `KeycloakJwtService` or `JwtService`, those must be available in `ProjectModule` — even if the guard is exported from `AuthModule`.

**Two options:**
1. Export all transitive dependencies from `AuthModule` (requires re-exporting modules, can't re-export providers not owned by the module)
2. **Preferred:** Keep guards thin, move all auth logic to a service, only export the service

## Dependency Rules

- A guard exported from `AuthModule` and used in `ProjectModule` must only depend on:
  - Global providers (`Reflector`)
  - Providers exported from `AuthModule` (e.g., `UserService`)
- `UserService` can depend on `AuthService`, `KeycloakJwtService`, `JwtService` — these are internal to `AuthModule`
- `AuthModule` imports `KeycloakJwtModule` and `JwtModule` internally; these don't need to be exported if no guard depends on them directly

## TokenValidationResult / UserContext

Use a single `UserContext` interface as the auth result contract, defined in `user.service.ts`:

```typescript
// user.service.ts — single source of truth
export interface UserContext {
  userId?: string
  adminPermissions?: bigint
}
```

`TokenValidationResult` in `auth.service.ts` should have matching field names:

```typescript
// auth.service.ts
export interface TokenValidationResult {
  userId: string        // NOT 'id' — must match UserContext.userId
  adminPermissions: bigint
}
```

**Field name consistency is critical.** If `auth.service.ts` uses `userId` but `keycloak-jwt.service.ts` returns `id`, the guard silently gets `undefined`. Always verify all services return the same field names.

## Migration Steps (from old pattern)

1. Create `UserService` in `auth/user.service.ts` — `authenticate(request)` handles both DSO token and Bearer JWT
2. Create `UserGuard` — thin guard that only calls `userService.authenticate()` and attaches result to request
3. Rewrite `AdminPermissionGuard` — calls `userService.authenticate()` then checks permissions via `Reflector`
4. Remove direct `JwtService`/`KeycloakJwtService` dependencies from both guards
5. Update all specs — guard specs only mock `UserService` + `Reflector`, auth flow tests belong in `UserService` spec
6. Update `AuthModule` — add `UserService` to providers and exports

## Common Pitfalls

- **Don't export a provider from a module that doesn't own it** — NestJS throws `UnknownExportException`.
- **Don't add a module to `exports` if it's only needed internally** — it creates unnecessary coupling.
- **Keep `UserContext` in one place** — import the interface from `UserService` everywhere.
- **mockDeep doesn't work for NestJS provider injection** — Use plain object mocks: `{ provide: UserService, useValue: { authenticate: vi.fn() } }`
- **Catch JWT verification errors** — wrap `jwtService.verifyAsync()` in try/catch, re-throw as `UnauthorizedException`.
- **Avoid dynamic `import()` types in `Reflector.get`** — causes runtime `TypeError`. Import types statically.
