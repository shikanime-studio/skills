---
name: nestjs-build-fixing
description:
  "Fix NestJS + TypeScript build errors and migrate auth/routes from legacy
  Fastify to NestJS: Zod schema wiring, JWT validation with @nestjs/jwt, JWKS
  caching, decorator resolution, Prisma mocking, server migration patterns.
  Triggered when user commands 'fix build', 'migrate [resource]', 'add auth', or
  when TS compilation errors appear in server-nestjs."
---

# NestJS Build Fixing & Auth Migration

Fix TypeScript compilation errors and migrate auth modules in NestJS projects
(especially `apps/server-nestjs`).

## Procedure

1. **Build first, read ALL errors**

   ```bash
   pnpm build 2>&1 | grep 'error TS' | sed 's/^.*error //' | sort -u
   ```

   Do not just `tail` — errors at the top reveal root causes. Categorize:
   missing names, wrong import paths, type vs value issues.

2. **Trace missing names to their actual source**
   - `rg -l 'ExportName' src/` to find which file exports the symbol
   - Distinguish `import type` (compile-time) vs `import` (runtime value).
     Errors like "cannot be used as a value because it was imported using
     `import type`" mean switch to plain import.
   - Verify actual directory layout with `find src -type d` — NestJS paths are
     often 3-4 levels deep.

3. **Fix import paths systematically**
   - Read the actual source files that export the needed symbols
   - Build the complete import block — add ALL missing imports in one pass
   - For `@cpn-console/shared` imports: use
     `rg 'export.*Name' packages/shared/src/` to find the export location

4. **Fix logic bugs exposed by the build**
   - Undefined variables, wrong method calls, mismatched parameter counts or
     types

5. **Verify with build + lint + test**

   ```bash
   pnpm build 2>&1 | tail -5
   pnpm lint 2>&1 | tail -5
   pnpm test 2>&1 | tail -10
   ```

## Pitfalls

- **`import type` vs `import`**: `ZodValidationPipe(schema)` needs the schema as
  a runtime value. If imported with `import type`, TS1361 fires. Split: types in
  `import type`, values in plain `import`.
- **NestJS module paths are nested 3-4 levels deep**: Always verify with `find`
  rather than guessing.
- **Shared package contracts**: `projectContract` from `@cpn-console/shared` is
  a runtime value (ts-rest router object). Import as value, not type.
- **`@nestjs/common` re-exports**: `UnauthorizedException`,
  `ForbiddenException`, etc. come from `@nestjs/common`, NOT from
  `@nestjs/core`.
- **`patch` tool matches on stale content**: After multiple edits, `patch` may
  report "found N matches" or match on old content. Re-read the file before
  patching.
- **Dual-server port conflict**: The legacy server (`apps/server`) and NestJS
  server (`apps/server-nestjs`) must NOT share the same `SERVER_PORT`. Legacy
  uses 4001, NestJS uses 3001. If NestJS has `SERVER_PORT=4000`, it grabs the
  client's default port and all requests hit NestJS, causing 404s on
  non-migrated routes. Check with `lsof -i -P -n | grep node.*LISTEN`. See
  `references/cpn-console-nestjs.md` for the full port matrix and
  nginx-strangler routing setup.
- **Nginx-strangler route registration**: When migrating a new module to NestJS,
  you MUST add its path to `apps/nginx-strangler/conf.d/routing.conf` and reload
  nginx (`docker compose exec nginx-strangler nginx -s reload`). Without this,
  requests to the new route still hit the legacy server which no longer has the
  handler.
- **Dual-auth guard pattern**: Guard checks `x-dso-token` first, then
  `Authorization: Bearer`. Both are first-class — `x-dso-token` is NOT "legacy".
  Route ALL JWT verification through `@nestjs/jwt` `JwtService` — never
  duplicate JWT crypto. Use async `secretOrKeyProvider` for JWKS key resolution.
- **Use official NestJS libraries**: `@nestjs/jwt` for JWT,
  `@nestjs/cache-manager` for caching, `@nestjs/common` `Logger` for logging.
  Don't hand-roll.
- **`decodeJweHeader` accepts union type**: When called from
  `secretOrKeyProvider`, `tokenOrPayload` is
  `string | object | Buffer<ArrayBufferLike>`. Handle non-string by returning
  `null`.
- **Zod at trust boundaries**: Validate ALL untrusted external data (JWKS
  responses, JWT payloads, decoded headers) with Zod schemas. Never use `as`
  casts. Use `safeParse` + explicit `!result.success` check — `parse` throws
  `ZodError` that bypasses NestJS exception filters.
- **Zod `.preprocess` for coercion**: When untrusted data may have wrong types,
  use `z.preprocess(coerceFn, schema)`:

  ```ts
  const coerceToString = (val: unknown) => (typeof val === "string" ? val : "");
  z.preprocess(coerceToString, z.string().default(""));
  ```

- **Split JWKS into its own module**: To avoid circular dependencies between
  `AuthModule` and the JWT service, create a separate `KeycloakJwtModule` that
  provides `KeycloakJwtService`. `AuthModule` imports `KeycloakJwtModule`.
- **`CacheModule.register()` defaults**: Default in-memory store uses TTL in
  milliseconds. Make cache TTLs configurable via `ConfigurationService` (e.g.,
  `keycloakJwksCacheTtlMs = Number(process.env.KEYCLOAK_JWKS_CACHE_TTL_MS ??
  300_000)`)
  rather than hardcoding them. Use a named constant `JWKS_CACHE_TTL_MS` in the
  service to reference the config value.
- **Don't register `CacheModule` inside small shared modules**: Registering
  `CacheModule.register()` inside a small provider module (like
  `KeycloakModule`) that is imported by another module can create
  duplicate/wrong module-scoped cache instances and break dependency resolution
  or lead to unexpected cache isolation. Prefer registering cache at a
  higher-level module or in application bootstrap, then have small modules
  simply `@Inject(CACHE_MANAGER)` in services.
- **Faker in tests**: Use `@faker-js/faker` for all test data — no hardcoded
  UUIDs, emails, or tokens. Create `*-testing.utils.ts` factory files with
  `make<TypeName>(overrides?)` pattern following the project's existing `make*`
  style.
- **Keycloak subfolder**: Keycloak JWT service, module, schema, and testing
  utils live in `auth/keycloak/` subfolder, not directly in `auth/`.
- **`secretOrKeyProvider` returns PEM string**:
  `createPublicKey({ key: { kty: 'RSA', n, e }, format: 'jwk' }).export({
  format: 'pem', type: 'pkcs1' })`
  produces the right format.
- **Prisma mockDeep strict types**: `mockDeep<PrismaService>()` generates strict
  return types that reject partial objects. Never use `as any` on mock return
  values. Instead, create typed factory helpers in `*-testing.utils.ts` (e.g.,
  `makeMockUser({...})`, `makeMockAdminRole({...})`) that return properly typed
  objects with faker defaults. See `references/testing-nestjs-prisma.md` and
  `references/testing-utils-factory-style.md`.
- **Prisma mock shapes must include ALL required fields**:
  `mockDeep<PrismaService>()` enforces strict types. For `PersonalAccessToken`,
  required fields include `id`, `name`, `status`, `expirationDate` (non-nullable
  `DateTime`), `lastUse`, `createdAt`, `hash`, `userId`, and the `owner`
  relation (included via `include: { owner: true }`). For `AdminToken`,
  `expirationDate` is nullable (`DateTime?`) but
  `PersonalAccessToken.expirationDate` is NOT nullable. Always check the Prisma
  schema for nullability before setting mock values. Use `as any` only as a last
  resort when the mock shape is deeply nested with relations.
- **Keep the auth flow simple: `authenticateHeaders` returns `UserContext` with
  `adminPermissions` already resolved**: For DSO tokens, `authenticateDsoToken`
  internally validates the token AND resolves admin permissions in one sequence
  — no `tokenResult` or `tokenKind` on `UserContext`. For Bearer JWTs,
  `keycloakJwtService.validatePayload` already returns resolved permissions. The
  guard just calls `authenticateHeaders()` then `AdminService.validate()`. No
  guard-level permission re-fetching needed.
- **When a rebase changes auth shape, update all consumers in one pass**: If
  `UserContext`, `RequestWithUserContext`, or request mocks change, patch the
  guard, helper factory, and all affected specs together. A half-migrated tree
  tends to leave stale `adminPermissions` assertions,
  `Property ... does not exist` TS errors, or failed `BigInt` coercions. See
  `references/auth-request-shape-split.md`.
- **Post-rebase admin-role wiring**: After rebasing admin-related modules to new
  auth, legacy controllers often still import obsolete `UserGuard` +
  `AdminPermissionGuard`/`AdminPermissionGuard`. The current auth stack uses
  only `AdminGuard` from `infrastructure/auth/admin/admin.guard` plus
  `@RequireAdminPermission()` from
  `infrastructure/auth/admin/admin-permission.decorator`. Before declaring
  success, grep for stale `AdminPermissionGuard`, `UserGuard`, and
  `admin-permission.guard` in the affected controller path and reconcile
  imports. `tsc --noEmit` catches unresolved modules and unused imports.
- **Admin permission bit values in tests**: `AdminAuthorized` functions check
  against `ADMIN_PERMS` bit positions. Key values: `Manage = bit(1) = 2n`,
  `List = bit(0) = 1n`, `ListSystem = bit(15) = 32768n`,
  `ManageSystem = bit(8) = 256n`. Using the wrong bit value (e.g., `32768n` for
  `Manage`) causes `ForbiddenException` at runtime. Always verify against
  `ADMIN_PERMS` in `packages/shared/src/utils/permissions.ts`.
- **Audit dependencies when moving methods**: When moving a method from one
  service to another (e.g., `validateKeycloakPayload` from `AuthService` to
  `KeycloakJwtService`), check the target service's constructor for new
  dependencies. Update all specs that instantiate the target service to provide
  those dependencies in `Test.createTestingModule`. A common failure is
  `Error: Nest can't resolve dependencies of the [Service] (...)`.
- **When resolving conflicts, fix downstream test failures in the same pass**:
  Conflict resolution (git or Jujutsu) often leaves type mismatches, renamed
  fields, or incorrect test data in the resolved files. After resolving conflict
  markers, always run `tsc --noEmit` and the relevant tests, then fix any
  failures before considering the resolution complete. Common patterns: renamed
  interface fields (`userId` → `id`), new service constructor dependencies
  missing from test modules, incorrect bit values in permission tests.
- **Jujutsu conflict markers differ from git**: Jujutsu uses
  `<<<<<<< conflict N of N`, `+++++++ revision "message" (destination)`,
  `%%%%%%% diff from: ...`, `\\\\\\\\\\\\ to: ...`,
  `>>>>>>> conflict N of N ends` — not the standard git `<<<<<<< HEAD`,
  `=======`, `>>>>>>>` format. Resolve by understanding which side's changes to
  keep, then write the clean file directly rather than trying to use `patch` on
  conflicted content.
- **Jujutsu workflow**: User primarily uses `jj` not `git`. Use `jj status` to
  check state, `jj log` for history. After resolving conflicts, `jj` auto-tracks
  file changes — no `git add` needed. The working copy is clean when `jj status`
  shows "The working copy has no changes."
- **write_file can be silently reverted**: In some cases, `write_file` reports
  success but the file on disk retains old content (observed during conflict
  resolution). Always verify with `cat` or `read_file` after writing, especially
  for critical conflict resolutions. If the file still has conflict markers,
  retry the write.
- **Guard constructor dependencies must ALL be mocked in specs**: When a guard
  injects multiple services (`AuthService`, `KeycloakJwtService`, `JwtService`,
  `Reflector`), every test module must provide ALL of them. NestJS error "can't
  resolve dependencies of the [Guard] (..., ?, ...)" means a mock is missing at
  the `?` index. Check the guard's constructor parameter order to identify which
  dependency is missing.
- **Extract authentication into a dedicated `AdminService`**: Don't put
  HTTP-level auth logic in `AuthService`. Create a separate `AdminService` with
  `authenticate(request)` that handles token extraction from headers and
  dispatches to `AuthService.validateToken()` (DSO token) or
  `JwtService.verifyAsync()` + `KeycloakJwtService.validatePayload()` (Bearer
  JWT). Guards then only depend on `AdminService`, keeping the dependency graph
  clean. See `references/guard-service-separation.md`.
- **Extract policy validation into three layers (core service → policy service →
  guard)**: After authentication, guards validate _policy_ (admin permissions,
  user types, project permissions/status/locked). Extract into three layers:
  1. **Core service** — pure business logic (`AdminService`, `ProjectService`).
     Method named `validate()` on both for symmetry.
  2. **Policy service** — builds the policy from reflector
     (`AdminPolicy.build()`, `ProjectPolicy.build()`). Depends on Reflector only
     (no core service dependency).
  3. **Guard** — thin orchestrator. Calls `policyService.build(context)`,
     delegates to core service for validation via `coreService.validate()`
     directly (NOT through policy service).

  **`resolveAdminPermissions` is private on `AuthService`, called from
  `authenticateDsoToken`**: The method that queries `adminRole.findMany` to
  merge global roles with token or user role permissions is a private method on
  `AuthService`. `authenticateDsoToken()` calls `validateToken()` to get the raw
  `TokenUserResult`, then calls `resolveAdminPermissions()` to resolve
  permissions, and returns `UserContext` with `adminPermissions` already set.
  This keeps the guard clean — it only calls `authenticateHeaders()` and passes
  the result to `AdminService.validate()`. No `tokenResult` or `tokenKind` field
  on `UserContext` — permissions are resolved at auth time, not deferred to the
  guard.

  **File layout**: Admin files go in `admin/`, project files in `project/`.
  Rename `AuthGuard` → `AdminGuard` and move it to `admin/`. File names use dot
  notation: `{domain}.{function}.ts` (e.g. `admin.guard.ts`, `admin.service.ts`,
  `project.locked.decorator.ts`). Folder names are `admin/` and `project/`.
  `user-type.decorator.ts` lives at `auth/` root level, not in `admin/`.
  `TokenUserResult` type lives in `auth.service.ts` (not in admin/). Update ALL
  imports, the module registration, and any `.overrideGuard(AuthGuard)` specs.
  - **Register ALL layers in `AuthModule` providers and exports. See
    `references/policy-service-extraction.md` for the full pattern
    (architecture, file-by-file spec, testing, migration checklist).**

  - **Decompose into sub-modules with `forwardRef` for circular dependencies**:
    Once the `admin/` and `project/` layers grow, extract them into their own
    NestJS modules (`AdminModule`, `ProjectModule`) and have `AuthModule` import
    them. Use `forwardRef(() => AuthModule)` in `AdminModule` when it injects
    `AuthService` (`AdminGuard` needs `AuthService`). `ProjectModule` typically
    doesn't need `forwardRef` — `ProjectGuard` only injects `ProjectService` and
    `ProjectPolicy`, both local to the module. Both sub-modules import
    `DatabaseModule` directly for `PrismaService`. `AuthModule` exports the
    sub-modules so downstream consumers can import `AuthModule` once and get
    access to all guards/services. Example:

    ```ts
    // admin/admin.module.ts
    @Module({
      imports: [forwardRef(() => AuthModule), DatabaseModule],
      providers: [AdminGuard, AdminService, AdminPolicy],
      exports: [AdminGuard, AdminService, AdminPolicy],
    })
    export class AdminModule {}

    // project/project.module.ts (no circular dep — ProjectGuard has no
      AuthService)
    @Module({
      imports: [DatabaseModule],
      providers: [ProjectGuard, ProjectService, ProjectPolicy],
      exports: [ProjectGuard, ProjectService, ProjectPolicy],
    })
    export class ProjectModule {}

    // auth.module.ts
    @Module({
      imports: [forwardRef(() => AdminModule), ProjectModule, ...],
      providers: [AuthService],
      exports: [AuthService, AdminModule, ProjectModule, ...],
    })
    export class AuthModule {}
    ```

    See `references/circular-module-deps.md` for the full pattern and migration
    steps.

- **`mockDeep<ExecutionContext>` corrupts complex return values**: When
  constructing test ExecutionContext objects for guard specs,
  `mockDeep<ExecutionContext>()` + `mockReturnValue(requestObject)` can strip
  complex properties (like `ProjectConfig` objects) from the returned request.
  The guard then sees `request.project === undefined` even though the test set
  it. **Fix**: construct `ctx` as a plain object with real arrow functions
  instead of using `mockDeep`:

  ```ts
  const ctx = {
    switchToHttp: () => ({
      getRequest: <T = any>() => request as T,
      getResponse: () => ({}) as any,
      getNext: () => (() => {}) as any,
    }),
    getHandler: () => (() => {}) as any,
    getClass: () => class {},
  } as ExecutionContext;
  ```

  The same issue applies when using `mockDeep<any>()` for `httpArgs` in a helper
  function. Use the plain-object pattern instead. This reliably preserves all
  properties on the request object including BigInt values, nested objects, and
  arrays.
- **`mockDeep` proxies break the `in` operator**: vitest-mock-extended creates
  ES Proxies for all nested properties. The `in` operator (e.g.
  `'adminRoleIds' in pat.owner`) returns `false` for properties that exist on
  the underlying object because the proxy's `has` trap doesn't forward to the
  target. **Fix**: replace `in` with `??` nullish coalescing:

  ```ts
  // DON'T: 'adminRoleIds' in pat.owner ? pat.owner.adminRoleIds : []
  // DO:
  ownerAdminRoleIds: pat.owner.adminRoleIds ?? [],
  ```

  Applies to any property-access check on objects returned by
  `mockResolvedValue`. The same mock value works fine for direct property access
  (`pat.owner.adminRoleIds` reads the correct value) — only the `in` operator is
  broken.
- **`resolveProjectPermissions` lives with the loader, not the validation
  service**: When `ProjectLoaderService` builds `ProjectConfig` from DB, it also
  owns `resolveProjectPermissions()`.
  `ProjectService.validateProjectPermissions()` delegates to
  `this.loader.resolveProjectPermissions()` — never duplicate the logic. Mirrors
  how `AuthService` owns both `validateToken()` and private
  `resolveAdminPermissions()`. The service that fetches data is the natural
  owner of resolution logic.
- **Requirements-based DB selects**: Pass a requirements object (derived from
  policy) to loader methods to conditionally include fields in Prisma `select`.
  Avoids over-fetching:\n

  ````ts
  interface ProjectRequirements {
    includeStatus?: boolean
    includeLocked?: boolean
    includePermissions?: boolean
  }\n
  function makeProjectSelect(requirements?: ProjectRequirements):
    Prisma.ProjectSelect {
    const perm = requirements?.includePermissions ?? true
    return {
      id: true, slug: true,
      ...(requirements?.includeStatus ? { status: true } : {}),
      ...(requirements?.includeLocked ? { locked: true } : {}),
      ...(perm ? { ownerId: true, everyonePerms: true, roles: { select: { id:
        true, permissions: true } }, members: { select: { userId: true, roleIds:
          true } } } : {}),
    } satisfies Prisma.ProjectSelect
  }\n
  // Guard derives requirements from policy and passes to loader
  const requirements = makeProjectRequirements(policy)
  request.project = await this.loader.load(request, requirements)
  ```\n
  Follow same pattern as `makeUserSelect`/`makeAdminTokenSelect` in
  `AuthService`.

  **Key consequences of the view-model pattern**:
  - `ProjectService` validates against `project.projectPermissions` directly —
    no longer injects `ProjectLoaderService`
  - `resolveProjectPermissions` is private on the loader; the validation service
    never calls it
  - Use `satisfies Prisma.ProjectSelect` on the return value (not `:
    Prisma.ProjectSelect` annotation) to catch typos while preserving the
    literal type for Prisma's `GetPayload` inference

  Follow same pattern as `makeUserSelect`/`makeAdminTokenSelect` in
  `AuthService`.

  **Key consequences of the view-model pattern**:
  - `ProjectService` validates against `project.projectPermissions` directly —
    no longer injects `ProjectLoaderService`
  - `resolveProjectPermissions` is private on the loader; the validation service
    never calls it
  - Use `satisfies Prisma.ProjectSelect` on the return value (not `:
    Prisma.ProjectSelect` annotation) to catch typos while preserving the
    literal type for Prisma's `GetPayload` inference

  ````

- **`mockDeep` doesn't work for NestJS provider injection**:
  `mockDeep<ServiceClient>()` creates a mock that doesn't get properly injected
  by NestJS testing module. Use plain object mocks with `vi.fn()` instead:
  `{ provide: ServiceClass, useValue: { method: vi.fn() } }`. This applies to
  all service mocks in guard specs.
- **Catch JWT verification errors**: `jwtService.verifyAsync()` throws plain
  `Error` on invalid JWTs, not `UnauthorizedException`. Always wrap in try/catch
  and re-throw as `UnauthorizedException` so NestJS exception filters handle it
  properly.
- **Avoid dynamic `import()` types in `Reflector.get`**: Using
  `keyof typeof import('@cpn-console/shared')` as a type parameter in
  `Reflector.get()` at runtime causes `TypeError` because the dynamic import
  isn't resolved. Import the type statically or restructure the guard to avoid
  the dynamic type in the reflector call.
- **Renaming guard files requires updating ALL references**: When renaming a
  guard file, also rename the class, the context interface, the decorator file,
  and update every import across guards, decorators, modules, specs, and any
  other files that reference them. Example: renaming `auth.guard.ts` →
  `admin/admin.guard.ts` requires `AuthGuard` → `AdminGuard`, plus updating
  `auth.module.ts`, `service-chain.controller.spec.ts`, and all
  `.overrideGuard()` references.
- **Keep schema and type colocated when the module is small**: For Keycloak
  payloads, it is acceptable to define `KeycloakPayloadSchema` and
  `KeycloakPayload` in `keycloak.service.ts` when the service is the natural
  boundary. Downstream code should import both from the service and avoid
  reaching into a separate schema file unless the schema is reused outside the
  service boundary.
- **Keycloak service spec dependency drift**: If `KeycloakJwtService` starts
  injecting `PrismaService`, its unit spec must provide a Prisma mock even if
  the tested methods do not touch the database directly.
- **JWKS refresh should log the unknown kid**: When `getPublicKey(kid)`
  encounters an uncached kid, log `Unknown kid "${kid}", refreshing JWKS` before
  calling `fetchJwks()`. This makes key rotation issues visible in logs.
- **JWKS integration test patterns**: Cover three scenarios in `getPublicKey`
  tests: (1) cache hit — kid already in cache, no fetch call; (2) cache miss —
  unknown kid triggers fetch, key is cached and returned; (3) key rotation — old
  kid cached, new kid arrives, fetch is called, new key cached, kids list
  replaced. Use `makeJwksResponse(kid)` from `keycloak-testing.utils.ts` to
  generate realistic JWKS responses.
- **When collapsing a schema file into a service, delete the stale file and
  update all imports in one pass**: this includes guards, test factories, and
  any Zod validation entry points that previously imported the schema directly.
- **Guard/service separation**: Guards should be thin — only depend on
  `Reflector` and exported services. All auth logic (token parsing, JWT
  verification, DB lookups) belongs in services. When a guard is used via
  `@UseGuards()` in a feature module, NestJS resolves its dependencies in the
  **consuming** module's scope, not the providing module's scope. If the guard
  directly injects `KeycloakService` or `JwtService`, those must be available in
  every module that uses the guard — which creates fragile cross-module
  coupling. **Fix:** move all auth logic into a service (e.g.,
  `AuthService.authenticateRequest()`), have the guard only call the service and
  check permissions. See `references/guard-service-separation.md`.
- **Don't collapse duplication by re-exporting across sibling modules**: If
  `KeycloakService` is provided by `KeycloakModule` and consumed by a guard in
  `AuthModule`, you can't re-export `KeycloakService` from `AuthModule` unless
  `AuthModule` lists it in its own `providers`. You can only export: (a)
  providers the module directly declares, or (b) providers that are exported
  from an imported module and explicitly re-exported. When the dependency is
  leaf-scoped, import that source module directly in any consumer feature module
  whose guards need it.

  **Do NOT add the provider to the exports array of a module that doesn't list
  it in its own providers** — NestJS throws `UnknownExportException` ("cannot
  export a provider/module that is not a part of the currently processed
  module").

## JWT / Auth Migration Pattern

When migrating Keycloak JWT validation from a legacy Fastify app to NestJS:

1. **Create `KeycloakJwtService`** in `auth/keycloak/` subfolder:
   - `getPublicKey(kid)` — fetches+caches JWKS from Keycloak certs endpoint
   - `decodeJweHeader(token)` — extracts `kid` from JWT header
   - `getIssuer()` — builds issuer URL from config
   - Uses `@nestjs/cache-manager` for TTL-based caching (no manual expiry
     tracking)

2. **Register `JwtModule.registerAsync`** with:
   - `secretOrKeyProvider: async (_type, token) => { const { kid } =
     decodeJweHeader(token); return getPublicKey(kid) }`
   - `verifyOptions: { algorithms: ['RS256'], issuer }`

3. **Guard uses `jwtService.verifyAsync(token)`** directly (not a separate
   `keycloakJwtService.validateJwt()`):

   ```ts
   const rawPayload = await this.jwtService.verifyAsync(jwt);
   const payload = KeycloakPayloadSchema.parse(rawPayload);
   ```

4. **Validate with Zod** — `KeycloakPayloadSchema` with `.preprocess` coercion
   on all optional fields. For small modules, the schema/type may live in the
   service file; keep imports pointed at that boundary.

5. **Sync OIDC roles** — In `validateKeycloakPayload`, merge OIDC group role IDs
   into user's stored `adminRoleIds` and update `lastLogin`

## NestJS Permission Decorator Pattern

See the existing skill content for `@RequireProjectPermission()` /
`ProjectPermissionGuard` patterns.

## Common NestJS Import Categories

| Category   | Typical Source                                                                        | Example                                             |
| ---------- | ------------------------------------------------------------------------------------- | --------------------------------------------------- |
| Decorators | `@nestjs/common`                                                                      | `Get`, `Put`, `Query`, `Body`, `Post`, `Controller` |
| Guards     | `modules/infrastructure/auth/admin/admin.guard.ts` or `auth/project/project.guard.ts` | `AdminGuard`, `ProjectGuard`                        |
| Schemas    | `@cpn-console/shared`                                                                 | `projectContract.*.body`, `ProjectSchemaV2`         |
| Services   | `./<module>.service.ts`                                                               | `ProjectService`                                    |
| Pipes      | `modules/infrastructure/pipe/*.pipe.ts`                                               | `ZodValidationPipe`                                 |

## Reference

See `references/auth-dso-flow-compact.md` for the DSO token auth flow — why
`resolveAdminPermissions` is private on `AuthService` and how the guard stays
clean.\nSee `references/cpn-console-nestjs.md` for project-specific paths and
patterns. See `references/testing-nestjs-prisma.md` for Prisma mocking patterns
in NestJS unit tests. See `references/testing-utils-factory-style.md` for the
preferred `make<TypeName>` test factory style. See
`references/jwks-testing-patterns.md` for JWKS cache hit/miss/rotation test
patterns, `makeJwksResponse` factory, configurable TTL setup, and
logging-on-miss guidance. See `references/guard-service-separation.md` for the
guard/service separation pattern — why guards must be thin and how to avoid
`UnknownDependenciesException` when guards are used across module boundaries.
See `references/policy-service-extraction.md` for the policy validation
extraction pattern — extracting admin/project policy checks into dedicated
services so the guard becomes a thin orchestrator. See
`references/auth-request-shape-split.md` for the request identity/permission
split used after auth rebase conflicts. See
`references/jujutsu-conflict-resolution.md` for Jujutsu conflict marker format,
resolution strategy, and workflow commands.
