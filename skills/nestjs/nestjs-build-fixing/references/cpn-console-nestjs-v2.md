# CPN Console NestJS — Project-Specific Reference

Paths and patterns for the `apps/server-nestjs` workspace in the cloud-pi-native
console monorepo.

## Directory Structure

```text
apps/server-nestjs/src/
├── modules/
│   ├── infrastructure/
│   │   ├── auth/
│   │   │   ├── user.guard.ts                   → UserGuard, UserContext,
  RequestWithUserContext
│   │   │   ├── user.decorator.ts                → @User()
│   │   │   ├── project.guard.ts                 → ProjectContextGuard,
  ProjectContext, RequestWithProjectContext
│   │   │   ├── project.decorator.ts             → @Project()
│   │   │   ├── project-status.guard.ts          → ProjectStatusGuard
│   │   │   ├── project-status.decorator.ts      → @RequireProjectStatus(...)
│   │   │   ├── project-locked.guard.ts          → ProjectLockedGuard
│   │   │   ├── project-permission.guard.ts      → ProjectPermissionGuard
│   │   │   ├── project-permission.decorator.ts  →
  @RequireProjectPermission(...)
│   │   │   ├── admin-permission.guard.ts        → AdminPermissionGuard
│   │   │   ├── admin-permission.decorator.ts    → @RequireAdminPermission(...)
│   │   │   ├── keycloak-jwt.service.ts          → KeycloakJwtService
  (JWKS-based JWT validation)
│   │   │   └── auth.module.ts                   → AuthModule (exports
  AuthService, KeycloakJwtService, guards)
│   │   ├── pipe/
│   │   │   └── zod-validation.pipe.ts           → ZodValidationPipe (wraps Zod
  schemas for NestJS)
│   │   └── database/
│   │       └── prisma.service.ts
│   ├── project/
│   │   ├── project.controller.ts                → All project + member routes
  (single controller)
│   │   ├── project.service.ts                   → All project + member business
  logic
│   │   ├── project.module.ts
│   │   ├── project.utils.ts
│   │   └── project-datastore.service.ts
│   └── ... (other modules)
```

## Key Import Patterns

```typescript
// TYPE imports (no runtime value needed)
import type { UserContext } from "../infrastructure/auth/user.guard";
import type { ProjectContext } from "../infrastructure/auth/project.guard";
import type { CreateProjectBody, Member, ProjectV2 } from "@cpn-console/shared";

// VALUE imports (needed at runtime for decorators/guards/pipes)
import { projectContract, projectMemberContract } from "@cpn-console/shared";
import { UserGuard } from "../infrastructure/auth/user.guard";
import { User } from "../infrastructure/auth/user.decorator";
import { ProjectContextGuard } from "../infrastructure/auth/project.guard";
import { Project } from "../infrastructure/auth/project.decorator";
import { ProjectPermissionGuard } from
  "../infrastructure/auth/project-permission.guard";
import { RequireProjectPermission } from
  "../infrastructure/auth/project-permission.decorator";

// Runtime schema extraction from ts-rest contracts
const ListProjectsQuerySchema = projectContract.listProjects.query; // runtime
  Zod schema
const BulkActionSchema = projectContract.bulkActionProject.body;
const UpdateProjectSchema = projectContract.updateProject.body;
```

## Permission Decorator Pattern

The NestJS server uses a two-tier admin/project permission system:

### Admin Permissions

- `@RequireAdminPermission(...)` + `AdminPermissionGuard` — checks
  `adminPermissions` bigint on the user context (set by `UserGuard`)
- Used for: admin-only routes (token management, project creation, CSV export)

### Project Permissions

- `@RequireProjectPermission(...)` + `ProjectPermissionGuard` — checks
  `projectPermissions` (set by `ProjectContextGuard`) against
  `ProjectAuthorized` helpers, with admin fallback
- Guard reads `adminPermissions` from `request.user` and `projectPermissions`
  from `request.project`
- Validates:
  `ProjectAuthorized[permName]({ adminPermissions, projectPermissions })`
- Available permission keys: `Manage`, `ListMembers`, `ManageMembers`,
  `ListRoles`, `ManageRoles`, `ListEnvironments`, `ManageEnvironments`,
  `ListRepositories`, `ManageRepositories`, `SeeSecrets`, `ReplayHooks`

### Self-removal bypass pattern

When a controller action should be allowed for either `ManagePermission` OR the
current user themselves (e.g., `removeMember`), do NOT put
`@RequireProjectPermission` on the controller. Instead, let the service handle
the full check including the self-bypass:

```typescript
// Controller — no permission guard, just status/locked guards
@Delete('/:projectId/members/:userId')
@UseGuards(UserGuard, ProjectContextGuard, ProjectStatusGuard,
  ProjectLockedGuard)
async removeMember(...) { ... }

// Service — full permission check with self-bypass
async removeMember(projectId, userId, requestorUserId, adminPermissions) {
  const state = await this.getProjectPermissionState(projectId, requestorUserId)
  if (!ProjectAuthorized.ManageMembers(...) && userId !== requestorUserId) {
    throw new ForbiddenException()
  }
  ...
}
```

## Server Migration Pattern (Fastify → NestJS)

When migrating a resource module from `apps/server` (Fastify) to
`apps/server-nestjs` (NestJS):

1. **Read the legacy router** (`apps/server/src/resources/<resource>/router.ts`)
   — identifies routes, contracts, permission checks
2. **Read the legacy business**
   (`apps/server/src/resources/<resource>/business.ts`) — core logic to port
3. **Read the shared contract** (`packages/shared/src/contracts/<resource>.ts`)
   — Zod schemas and ts-rest contract
4. **Add methods to NestJS service** — port business logic, replace:
   - `hook.xxx()` calls → `eventEmitter.emitAsync('eventName', payload)`
   - `ProjectAuthorized.xxx()` checks → keep in service for self-bypass cases,
     or move to `@RequireProjectPermission()` decorator
   - `logViaSession()` → handled by `KeycloakJwtService` +
     `AuthService.validateKeycloakPayload()` (JWT) or
     `AuthService.validateToken()` (token)
   - `logViaToken()` → handled by `AuthService.validateToken()`
5. **Add controller routes with decorator-based permissions** — prefer
   `@RequireProjectPermission()` over inline checks
6. **Register controller in module** — add to `controllers[]` in the module
7. **Update migration docs** — `MODULARISATION-STATUT.md` and
   `MODULARISATION-CARTOGRAPHIE.md`

### Controller consolidation preference

Prefer a single controller per resource domain (e.g., `ProjectController`
handles both `/projects` and `/projects/:projectId/members`) rather than
separate small controllers.

### EventEmitter as hook replacement

The legacy `hook.xxx()` system is replaced by `eventEmitter.emitAsync()`:

```typescript
// Legacy
await hook.projectMember.upsert(projectId, userId);

// NestJS
await this.eventEmitter.emitAsync("projectMember.upsert", {
  projectId,
  userId,
});
```

## Shared Package (`packages/shared`)

- `projectContract` — ts-rest router object. **Runtime value**, NOT just a type.
- `projectMemberContract` — ts-rest router for member routes. Runtime value.
- `ProjectSchemaV2` — Zod schema for V2 project objects. Value import.
- `ProjectV2`, `Member`, `CreateProjectBody` — Zod inferred types. Type imports.
- `PROJECT_PERMS`, `ADMIN_PERMS` — bitmask constant objects. Value imports.
- `ProjectAuthorized` — authorization helper functions `(params) => boolean`.
  Used in guards and services.
- `AdminAuthorized` — authorization helper functions `(perms) => boolean`. Used
  in guards.

## Build + Test + Lint Commands

```bash
pnpm build 2>&1 | tail -5    # quick status — "Done" = success
pnpm lint 2>&1 | tail -5     # quick status
pnpm test 2>&1 | grep -E '(Test Files|Tests |FAIL)' | grep -v '✓'
```

## Nginx Strangler

Migrated routes are routed to server-nestjs via the nginx strangler config at
`apps/nginx-strangler/conf.d/routing.conf`. When a new module is migrated, the
Nginx config must be updated to point its routes to the NestJS backend.

## Keycloak JWT Auth

Two auth methods (token header takes precedence):

1. **`x-dso-token` header** — SHA-256 hashed → lookup in `personalAccessToken` /
   `adminToken` tables via `AuthService.validateToken()`
2. **`Authorization: Bearer \*** — Keycloak JWT → `@nestjs/jwt`
   `JwtService.verifyAsync()` with async `secretOrKeyProvider` fetching JWKS
   public keys

### Architecture

```text
AdminPermissionGuard
  ├─ x-dso-token → AuthService.validateToken()
  └─ Bearer *** JwtService.verifyAsync() → AuthService.validateKeycloakPayload()
```

### KeycloakJwtService (subfolder: `auth/keycloak/`)

- Organized in `modules/infrastructure/auth/keycloak/` subfolder (keycloak
  JwtModule + service)
- **JWKS fetching**: Uses `@nestjs/cache-manager` with 5-minute TTL — NOT manual
  `Map`
- **Logger**: Uses `@nestjs/common` `Logger` — NOT `console.log`
- **`decodeJweHeader(token: string | object | Buffer)`**: Extracts `kid` from
  JWT header. Accepts the union type from `secretOrKeyProvider` callback.
  Returns `null` for non-string input.
- **`getPublicKey(kid)`**: Returns cached PEM public key or fetches from JWKS
- **`getPublicKey` is separate from decode** — the module's
  `secretOrKeyProvider` composes both

### JwtModule configuration

Use `JwtModule.registerAsync` with an async `secretOrKeyProvider`:

```typescript
JwtModule.registerAsync({
  imports: [ConfigurationModule, KeycloakJwtModule],
  inject: [ConfigurationService, KeycloakJwtService],
  useFactory: (config, keycloakJwtService) => ({
    secretOrKeyProvider: async (_requestType, tokenOrPayload) => {
      const decoded = keycloakJwtService.decodeJweHeader(tokenOrPayload);
      if (!decoded?.kid) throw new Error("Missing kid");
      const publicKey = await keycloakJwtService.getPublicKey(decoded.kid);
      if (!publicKey) throw new Error("Unknown signing key");
      return publicKey;
    },
    verifyOptions: {
      algorithms: ["RS256"],
      issuer:
       
          `${config.keycloakProtocol}://${config.keycloakDomain}/realms/${config.keycloakRealm}`,
    },
  }),
});
```

Key points:

- `secretOrKeyProvider` must be **async** — synchronous `.verify()` throws by
  design in `@nestjs/jwt` v11
- `decodeJweHeader` accepts `string | object | Buffer` — non-strings return
  `null`
- JWKS keys cached via `@nestjs/cache-manager`, not manual `Map` + TTL
- Issuer validation handled by `JwtService` via `verifyOptions.issuer`
- Do NOT duplicate JWT verification logic in the guard — route everything
  through `JwtService`

### Zod validation at trust boundaries

All untrusted inputs are validated with Zod before use — **never use `as` casts
on untrusted data**:

```typescript
const JwksResponseSchema = z.object({
  keys: z.array(
    z.object({
      kid: z.string(),
      kty: z.string(),
      use: z.string(),
      n: z.string(),
      e: z.string(),
    }),
  ),
});
const jwks = JwksResponseSchema.parse(raw); // throws on malformed data
```

Apply this to: JWKS responses, JWT payloads, decoded JWT headers — any data from
external sources.

### Node.js `createPublicKey` gotcha

```typescript
// WRONG — TS2769
key.export({ format: "pem" });

// CORRECT
key.export({ format: "pem", type: "pkcs1" });
```

### Testing: faker + factory functions

Use `@faker-js/faker` for all test data — no hardcoded values like `'user-1'`,
`'test@test.com'`, `'bad-token'`.

Create `*-testing.utils.ts` factory files per module. The
`make<TypeName>(overrides?)` pattern:

```typescript
export function makeKeycloakPayload(
  overrides: Partial<KeycloakPayload> = {},
): KeycloakPayload {
  return {
    sub: faker.string.uuid(),
    email: faker.internet.email().toLowerCase(),
    given_name: faker.person.firstName(),
    family_name: faker.person.lastName(),
    groups: Array.from(
      { length: faker.number.int({ min: 0, max: 3 }) },
      () => `/${faker.word.noun()}`,
    ),
    ...overrides,
  };
}
```

- Use `satisfies` for type-safe mock objects at factory boundaries
- Prisma `mockDeep` return values need `as any` cast — strict return types don't
  match partial mocks
- Guard tests: mock `JwtService.verifyAsync` (from `@nestjs/jwt`), not a custom
  JWT service
- Token values in tests: `faker.string.alphanumeric(16)` instead of
  `'bad-token'`

## Known Limitations (as of 2026-06-08)

- `hook.user.retrieveUserByEmail` not available — add-member by email returns an
  error.
- `DELETE /api/v1/projects/:projectId` (archive) exists in service but is not
  exposed as a separate controller route.
- `PUT /api/v1/projects/:projectId/hooks` (replay-hooks) exists in service but
  not in controller.
