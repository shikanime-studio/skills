# CPN Console NestJS — Project-Specific Reference

Paths and patterns for the `apps/server-nestjs` workspace in the cloud-pi-native
console monorepo.

## Directory Structure

```text
apps/server-nestjs/src/
├── modules/
│   ├── infrastructure/
│   │   ├── auth/
│   │   │   ├── user.guard.ts          → UserGuard, UserContext,
  RequestWithUserContext
│   │   │   ├── user.decorator.ts      → @User()
│   │   │   ├── project.guard.ts       → ProjectContextGuard, ProjectContext,
  RequestWithProjectContext
│   │   │   ├── project.decorator.ts   → @Project()
│   │   │   ├── project-status.guard.ts           → ProjectStatusGuard
│   │   │   ├── project-status.decorator.ts       → @RequireProjectStatus(...)
│   │   │   ├── project-locked.guard.ts           → ProjectLockedGuard
│   │   │   ├── project-permission.guard.ts       → ProjectPermissionGuard
│   │   │   ├── project-permission.decorator.ts   →
  @RequireProjectPermission(...)
│   │   │   ├── admin-permission.guard.ts         → AdminPermissionGuard
│   │   │   └── admin-permission.decorator.ts     → @RequireAdminPermission(...)
│   │   ├── pipe/
│   │   │   └── zod-validation.pipe.ts → ZodValidationPipe (wraps Zod schemas
  for NestJS)
│   │   └── database/
│   │       └── prisma.service.ts
│   ├── project/
│   │   ├── project.controller.ts
│   │   ├── project.service.ts
│   │   ├── project.module.ts
│   │   └── project.utils.ts
│   └── ... (other modules)
```

## Key Import Patterns file

```typescript
// TYPE imports (no runtime value needed)
import type { UserContext } from '../infrastructure/auth/user.guard'
import type { ProjectContext } from '../infrastructure/auth/project.guard'
import type { CreateProjectBody, ProjectV2 } from '@cpn-console/shared'

// VALUE imports (needed at runtime for decorators/guards/pipes)
import { projectContract, ProjectSchemaV2 } from '@cpn-console/shared'
import { UserGuard } from '../infrastructure/auth/user.guard'
import { User } from '../infrastructure/auth/user.decorator'
import { ProjectContextGuard } from '../infrastructure/auth/project.guard'
import { Project } from '../infrastructure/auth/project.decorator'

// Runtime schema extraction from ts-rest contracts
const CreateProjectSchema = ProjectSchemaV2.pick({ name: true, ... })
const ListProjectsQuerySchema = projectContract.listProjects.query   // runtime
  Zod schema
const BulkActionSchema = projectContract.bulkActionProject.body
const UpdateProjectSchema = projectContract.updateProject.body
```

## Shared Package (`packages/shared`)

- `projectContract` — ts-rest router object. **Runtime value**, NOT just a type.
- `projectMemberContract` — ts-rest router for member routes. **Runtime value**.
- `adminRoleContract` — ts-rest router for admin role routes. **Runtime value**.
- `ProjectSchemaV2` — Zod schema for V2 project objects. Value import.
- `ProjectV2` — Zod inferred type. Type import.
- `AdminRole` — Zod inferred admin role type (permissions as `string`). Type
  import.
- `Member` — Zod inferred member type. Type import.
- `CreateProjectBody` — ts-rest inferred body type. Type import.

## Controller Organization

Sub-resource routes (e.g., `/:projectId/members`) that share the same parent
path and guards should be merged into the parent controller. Create separate
controller classes only for truly independent modules with their own root path.

## Build Command

```bash
pnpm build 2>&1 | tail -5    # quick status
pnpm lint 2>&1 | tail -5     # quick status
```

## Nginx Strangler — Dual-Server Routing

The project uses a **strangler fig pattern**: nginx sits in front of both the
legacy Fastify server (`apps/server`) and the NestJS server
(`apps/server-nestjs`), routing requests based on path.

### Port Configuration (CRITICAL — avoid port conflicts)

| Server                        | Port                          | Config                                                                                   |
| ----------------------------- | ----------------------------- | ---------------------------------------------------------------------------------------- |
| Legacy (`apps/server`)        | 4001                          | `SERVER_PORT=4001` in `apps/server/.env`                                                 |
| NestJS (`apps/server-nestjs`) | 3001                          | `SERVER_PORT=3001` in `apps/server-nestjs/.env`                                          |
| nginx-strangler               | 8082 (Docker exposes as 4000) | `LEGACY_UPSTREAM=host.docker.internal:4001`, `NESTJS_UPSTREAM=host.docker.internal:3001` |
| Client Vite proxy target      | 8082                          | `SERVER_PORT=8082` in `apps/client/.env`                                                 |

**Both servers must NOT share the same port.** If NestJS has `SERVER_PORT=4000`,
it will collide with the legacy server and receive all traffic, causing 404s on
non-migrated routes.

**Diagnosing port conflicts:**

```bash
lsof -i -P -n | grep -E 'node.*LISTEN'    # See which node process owns each
  port
```

**Symptom:** All requests land on NestJS, non-migrated routes (e.g.,
`/api/v1/health-services`) return 404.

### Nginx Route Config

File: `apps/nginx-strangler/conf.d/routing.conf`

Currently migrated to NestJS:

- `/api/v1/service-chains`
- `/api/v1/deployments`
- `/api/v1/projects`

All other `/api/*` routes fall through to `location /api/` → `server-legacy`.

### Adding a New Migrated Module

1. Add the route block to `apps/nginx-strangler/conf.d/routing.conf`:

   ```nginx
   location /api/v1/new-resource {
       proxy_pass         http://server-nestjs;
       proxy_http_version 1.1;
       proxy_set_header   Host              $host;
       proxy_set_header   X-Real-IP         $remote_addr;
       proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
       proxy_set_header   X-Forwarded-Proto $scheme;
   }
   ```

2. Reload nginx: `docker compose exec nginx-strangler nginx -s reload`
3. Restart the NestJS server so it picks up the new module
4. Verify: `curl http://localhost:8082/api/v1/new-resource`

### Prisma Generated Types — Permission Fields

Prisma 6 generates `AdminRole.permissions` as `bigint`. The shared API contract
(`AdminRole` from `@cpn-console/shared`) uses `string`. The NestJS service must
convert:

```typescript
// In *.utils.ts mapper
role.permissions.toString(); // bigint → string
```

In tests: Prisma fixtures use `bigint` (`permissions: 4n`), expected API output
uses strings (`permissions: '4'`).

To inspect generated Prisma types:

```bash
rg 'permissions.*bigint'
  node_modules/.pnpm/@prisma+client*/node_modules/.prisma/client/index.d.ts
```
