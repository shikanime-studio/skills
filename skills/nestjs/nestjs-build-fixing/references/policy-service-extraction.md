# Policy Validation Extraction Pattern

## Architecture

After authentication, guards validate _policy_ — admin permissions, user types,
project permissions/status/locked.

### Three-Layer Split

```text
┌─────────────────────────────────────────────┐
│ admin/                                       │
│  admin.guard.ts   — thin orchestrator         │
│  admin-policy.service.ts — build policy from reflector│
│  admin.service.ts — pure logic (resolve+validate)│
│  admin.{permission,testing}.ts                │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ project/                                      │
│  project.guard.ts   — thin orchestrator       │
│  project.policy.ts  — build policy from reflector│
│  project.service.ts — pure logic (load+validate)│
│  project.{locked,permission,status,testing}.ts│
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ auth/ (root)                                 │
│  auth.service.ts — authentication + TokenUserResult type
│  auth.module.ts  — registers all services    │
│  auth-testing.utils.ts — shared makeContext   │
│  user-type.decorator.ts — RequireUserType     │
│  user.decorator.ts — @User() request param    │
│  keycloak-jwt/                               │
└─────────────────────────────────────────────┘
```

### Class Responsibilities

| Layer          | Class            | Responsibility                                                                                                                                                             | Dependencies                                        |
| -------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| Core service   | `AdminService`   | `validate(policy, user)` — check admin perms + user types. `resolveAdminPermissions(result)` — query `adminRole.findMany` to merge global roles with token/user role perms | `PrismaService`                                     |
| Core service   | `ProjectService` | `loadProject(request)`, `validate(policy, project, userId, perms)`, `resolveProjectPermissions(project, userId)`                                                           | `PrismaService`                                     |
| Policy service | `AdminPolicy`    | `build(context)` — read reflector metadata for admin perms + user types                                                                                                    | `Reflector` only                                    |
| Policy service | `ProjectPolicy`  | `build(context)` — read reflector for project perms, status, locked                                                                                                        | `Reflector` only                                    |
| Guard          | `AdminGuard`     | `build()` → `authenticateHeaders()` → `resolveAdminPermissions()` (if tokenResult) → `adminService.validate()`                                                             | `AuthService`, `AdminService`, `AdminPolicy`        |
| Guard          | `ProjectGuard`   | `build()` → `loadProject()` → `projectService.validate()`                                                                                                                  | `ProjectService`, `ProjectPolicy`                   |
| Auth           | `AuthService`    | `authenticateHeaders()`, `validateToken()` (returns raw `TokenUserResult`)                                                                                                 | `PrismaService`, `JwtService`, `KeycloakJwtService` |

### Key Rules

1. **`resolveAdminPermissions` lives in `AdminService`**, NOT in `AuthService`
   or `AdminPolicy`. It queries `adminRole.findMany` to merge global roles with
   token or user role permissions. `AdminPolicy` only owns `build(context)` —
   pure reflector operation with no Prisma dependency.

2. **`TokenUserResult` type lives in `auth.service.ts`**, alongside
   `UserContext` and `AuthRequirements`. It is the raw return type of
   `AuthService.validateToken()` before permission resolution. Imported by
   `AdminService` for `resolveAdminPermissions()` and by
   `UserContext.tokenResult`.

3. **`AuthService.validateToken()` returns raw `TokenUserResult`**, not resolved
   `UserContext`. Permission resolution is deferred to the guard which calls
   `adminService.resolveAdminPermissions()` from the raw `tokenResult` on
   `UserContext`.

4. **Guards call `coreService.validate()` directly**, not through the policy
   service. The policy service only builds the policy from reflector metadata —
   it does not own validation logic.

5. **Both `AdminService` and `ProjectService` use `validate()` as the method
   name.** This keeps the symmetry clean.

6. **`includeAdminRoleIds` controls DB query scope**, not
   `includeAdminPermissions`. This flag determines whether `adminRoleIds` is
   included in the Prisma owner select. Renamed from the old
   `includeAdminPermissions` to be explicit about what it controls.

7. **`user-type.decorator.ts` lives at `auth/` root level**, not inside `admin/`
   or `project/`, because `USER_TYPES_KEY` is a cross-cutting metadata key.

### File Naming Convention

- Folder names: `admin/`, `project/`
- File names use **dots** as separators: `{domain}.{function}.ts`
  - `admin.guard.ts`, `admin-policy.service.ts`, `admin.service.ts`
  - `project.guard.ts`, `project.policy.ts`, `project.service.ts`
  - `admin-permission.decorator.ts`, `project.locked.decorator.ts`
  - `project.status.decorator.ts`, `project.permission.decorator.ts`
  - `admin-testing.utils.ts`, `project.testing.utils.ts`
- `user-type.decorator.ts` lives at `auth/` root level
- NOTE: `admin-policy.service.ts` uses a hyphen (not `admin.policy.ts`) — this
  is inconsistent with the dot convention. The project side uses
  `project.policy.ts` with dots.

### Class Names

| Old Name                       | New Name                                                     |
| ------------------------------ | ------------------------------------------------------------ |
| `AdminService`                 | `AdminService` (unchanged, but was `UserService` for a time) |
| `AdminPolicyService`           | `AdminPolicy`                                                |
| `AdminGuard` (was `AuthGuard`) | `AdminGuard`                                                 |
| `AdminPolicy` (interface)      | `AdminPolicyConfig`                                          |
| `TokenAdminResult`             | `TokenUserResult` (lives in auth.service.ts)                 |
| `ProjectService`               | `ProjectService` (unchanged)                                 |
| `ProjectPolicyService`         | `ProjectPolicy`                                              |
| `ProjectContextGuard`          | `ProjectGuard`                                               |
| `ProjectPolicy` (interface)    | `ProjectPolicyConfig`                                        |
| `ProjectContext` (interface)   | `ProjectConfig`                                              |

### AuthRequirements Field Names

| Old Field                 | New Field             | Purpose                                                            |
| ------------------------- | --------------------- | ------------------------------------------------------------------ |
| `includeAdminPermissions` | `includeAdminRoleIds` | Controls whether `adminRoleIds` is included in Prisma owner select |

## AuthService Authentication Flow

```text
authenticateHeaders(headers)
  ├── authenticateDsoToken(headers)
  │     └── validateToken(rawToken, requirements)
  │           └── findAndValidateToken(hash) → TokenUserResult (raw, no
    resolution)
  │     returns UserContext { userId, userType, tokenResult }
  └── authenticateBearerToken(headers)
        └── keycloakJwtService.validatePayload(payload, requirements)
        returns UserContext { userId, adminPermissions, userType } (already
          resolved)

AdminGuard then:
  if (user.tokenResult && requirements.includeAdminRoleIds)
    user.adminPermissions = await
      adminService.resolveAdminPermissions(user.tokenResult)
  adminService.validate(policy, user)
```

`AdminService.resolveAdminPermissions(result)` queries
`prisma.adminRole.findMany` for global roles, then merges with either the admin
token's `permissions` field or the user's admin role IDs.

## Directory Layout

```text
auth/
├── admin/
│   ├── admin.guard.ts
│   ├── admin.guard.spec.ts
│   ├── admin-policy.service.ts     (+ AdminPolicyConfig interface)
│   ├── admin.service.ts            (+ resolveAdminPermissions)
│   ├── admin.service.spec.ts
│   ├── admin-testing.utils.ts
│   └── admin-permission.decorator.ts
├── project/
│   ├── project.guard.ts
│   ├── project.guard.spec.ts
│   ├── project.policy.ts
│   ├── project.service.ts
│   ├── project.testing.utils.ts
│   ├── project.locked.decorator.ts
│   ├── project.permission.decorator.ts
│   └── project.status.decorator.ts
├── auth.module.ts
├── auth.service.ts               (+ TokenUserResult type,
  UserContext.tokenResult)
├── auth.service.spec.ts
├── auth-testing.utils.ts
├── user-type.decorator.ts
├── user.decorator.ts
└── keycloak-jwt/
```

## Testing Notes

### ExecutionContext in Guard Specs

### ProjectConfig as a View Model (not raw DB shape)

`ProjectConfig` (returned by the loader) should expose **only what validators
and policy checks need**, not the raw Prisma model:

```ts
export interface ProjectConfig {
  id: string;
  slug: string;
  locked?: boolean; // only loaded if policy includes locked check
  status?: PrismaProject["status"]; // only loaded if policy includes status
  check;
  projectPermissions?: bigint; // resolved at load time — not raw role/member
  data;
}
```

**Removed from the interface**: `ownerId`, `everyonePerms`, `roles`, `members`.
These are internal to the loader's permission resolution and must not leak to
consumers.

`ProjectService.validateProjectPermissions()` reads `project.projectPermissions`
directly — it does NOT call back to the loader. This means `ProjectService` no
longer needs to inject `ProjectLoaderService` at all:

```ts
class ProjectService {
  validate(
    policy: ProjectPolicyConfig,
    project: ProjectConfig,
    userId: string,
  ) {
    // userId is for admin-override checks on the project
    const perms = project.projectPermissions;
    if (
      perms === undefined ||
      (policy.permission && !(policy.permission & perms))
    )
      throw new ForbiddenException();
  }
}
```

**Loading side** — `ProjectLoaderService.load()` accepts `userId`, resolves
permissions inline, and builds a slim `ProjectConfig`:

```ts
class ProjectLoaderService {
  async load(projectId: string, userId: string, requirements?:
    ProjectRequirements): Promise<ProjectConfig> {
    const project = await this.prisma.project.findUnique({
      where: { id: projectId },
      select: makeProjectSelect(requirements),
    })
    if (!project) throw new NotFoundException()
    return {
      id: project.id,
      slug: project.slug,
      ...(requirements?.includeStatus ? { status: project.status } : {}),
      ...(requirements?.includeLocked ? { locked: project.locked } : {}),
      ...(requirements?.includePermissions
        ? { projectPermissions: this.resolveProjectPermissions(project, userId)
          }
        : {}),
    }
  }

  private resolveProjectPermissions(project: ProjectWithRolesMembers, userId:
    string): bigint { ... }
}
```

### `satisfies` on Select Factories for Type Safety

Prisma `select` objects need to catch typos at compile time while preserving the
**literal type** for Prisma's `GetPayload` inference. Use `satisfies`:

```ts
function makeProjectSelect(requirements?: ProjectRequirements) {
  return {
    id: true,
    slug: true,
    ...(requirements?.includeStatus ? { status: true } : {}),
    ...(requirements?.includeLocked ? { locked: true } : {}),
    ...(requirements?.includePermissions
      ? {
          ownerId: true,
          everyonePerms: true,
          roles: { select: { id: true, permissions: true } },
          members: { select: { userId: true, roleIds: true } },
        }
      : {}),
  } satisfies Prisma.ProjectSelect;
}
```

`satisfies` validates the shape is assignable to `Prisma.ProjectSelect` but
preserves the literal type for inference. Without it, annotating
`: Prisma.ProjectSelect` would widen the type and break
`ProjectGetPayload<{ select: ... }>` return type inference.

## Testing Notes (continued)

### ExecutionContext in Guard Specs

**Do NOT use `mockDeep<ExecutionContext>()` for guard tests.** The `mockDeep`
proxy corrupts complex properties (like `ProjectConfig` objects, BigInt values,
nested objects) when passed through `mockReturnValue`. The guard sees
`request.project === undefined` even though the test set it.

**✅ Correct — plain object with real arrow functions:**

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

**❌ Incorrect — `mockDeep` proxies strip nested properties:**

```ts
const ctx = mockDeep<ExecutionContext>(); // BAD — project prop becomes
undefined;
const httpArgs = mockDeep<any>();
httpArgs.getRequest.mockReturnValue(request);
ctx.switchToHttp.mockReturnValue(httpArgs);
```

### Shared makeContext from auth-testing.utils

The shared `makeContext` from `auth-testing.utils.ts` uses `mockDeep` and works
for admin guard tests because the admin guard doesn't need to read complex
nested properties from the request — it only reads `userId`, `adminPermissions`,
and `userType` (primitive values). For project guard tests, use the plain-object
pattern above.

### Auth Service Tests

`validateToken()` now returns `TokenUserResult` (raw, no permission resolution).
Tests verify:

- Raw token shape (`kind`, `userId`, `ownerAdminRoleIds`/`permissions`,
  `userType`)
- `undefined` returned when no token found
- `UnauthorizedException` thrown for inactive/expired tokens
- `lastUse` and `lastLogin` updates
- `authenticateHeaders` passes raw `tokenResult` through
  `UserContext.tokenResult` for DSO tokens
- Keycloak bearer path works independently (permissions already resolved by
  `KeycloakJwtService`)
