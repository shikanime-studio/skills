# NestJS TypeScript Build Errors — Patterns & Fixes

## Context

When `nest build` (or `tsc`) fails in a NestJS project, the most common error
categories are:

1. **`Cannot find name 'X'` (TS2304)** — Missing import or wrong import path
2. **`cannot be used as a value because it was imported using 'import type'`
   (TS1361)** — `import type` used for a runtime value
3. **`Property 'X' is missing in type` (TS2345)** — Wrong schema/type used for a
   DTO

## Diagnostic Steps

### 1. Triage the error cluster

```bash
pnpm build 2>&1 | grep 'error TS'
```

Group by error code. A single missing import often cascades into 10+ errors —
fix the root cause first.

### 2. Fix TS2304 (Cannot find name)

For each missing name, determine:

- **NestJS decorator** (`@Get`, `@Put`, `@Query`, `@Body`)? → Add to
  `@nestjs/common` import
- **Custom decorator/guard** (`@User`, `@Project`, `UserGuard`)? →
  `rg "export.*ClassName" src/` to find the file, then import from the correct
  relative path
- **Type** (`UserContext`, `ProjectContext`)? → Import from the guard file that
  defines it

### 3. Fix TS1361 (import type used as value)

Split type-only from value imports:

```ts
// WRONG
import type { projectContract, CreateProjectBody } from "@cpn-console/shared";

// RIGHT
import type { CreateProjectBody } from "@cpn-console/shared";
import { projectContract, ProjectSchemaV2 } from "@cpn-console/shared";
```

### 4. Verify decorator-guard parity

Every custom decorator needs both a decorator file and a guard file, both
imported in the controller and registered in the module.

### 5. Schema/controller contract alignment

Derive Zod schemas from the shared contract to stay in sync:

```ts
const ListProjectsQuerySchema = projectContract.listProjects.query;
const BulkActionSchema = projectContract.bulkActionProject.body;
const UpdateProjectSchema = projectContract.updateProject.body;
```

Use `typeof Schema._type` for parameter type annotations.

## Case Study: project.controller.ts (2026-05-28)

**Symptom:** 51 `error TS` failures in `apps/server-nestjs`

**Root causes:**

1. Custom decorators/guards not imported at all
2. `projectContract` was `import type` but used as a runtime value
3. Standalone schema names that don't exist — should derive from
   `projectContract`
4. Dead `@Authenticated()` parameter left in handler
5. Handler called wrong service method (`createProject` vs `updateProject`)

**Fix:** Rewrote imports, derived schemas from `projectContract`, removed dead
code, fixed service call.
