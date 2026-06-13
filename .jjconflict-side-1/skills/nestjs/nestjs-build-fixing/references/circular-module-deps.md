# Circular Module Dependencies — forwardRef Pattern

When `AdminModule` (or `ProjectModule`) injects a service from `AuthModule` (e.g. `AdminGuard` injects `AuthService`), and `AuthModule` imports the child module, a circular dependency forms.

## Symptom

```
Nest cannot create the module instance.
A circular dependency has been detected (modules: AuthModule → AdminModule → AuthModule).
```

## Solution — `forwardRef`

### `admin/admin.module.ts`

```ts
import { forwardRef, Module } from '@nestjs/common'
import { DatabaseModule } from '../../database/database.module'
import { AuthModule } from '../auth.module'
import { AdminGuard } from './admin.guard'
import { AdminPolicy } from './admin-policy.service'
import { AdminService } from './admin.service'

@Module({
  imports: [
    forwardRef(() => AuthModule),  // ← circular dep
    DatabaseModule,                // ← PrismaService
  ],
  providers: [AdminGuard, AdminService, AdminPolicy],
  exports: [AdminGuard, AdminService, AdminPolicy],
})
export class AdminModule {}
```

### `project/project.module.ts`

No `forwardRef` needed — `ProjectGuard` only injects `ProjectService` and `ProjectPolicy`, both local to the module:

```ts
import { Module } from '@nestjs/common'
import { DatabaseModule } from '../../database/database.module'
import { ProjectGuard } from './project.guard'
import { ProjectPolicy } from './project.policy'
import { ProjectService } from './project.service'

@Module({
  imports: [DatabaseModule],
  providers: [ProjectGuard, ProjectService, ProjectPolicy],
  exports: [ProjectGuard, ProjectService, ProjectPolicy],
})
export class ProjectModule {}
```

### `auth.module.ts`

```ts
import { forwardRef, Module } from '@nestjs/common'
import { AdminModule } from './admin/admin.module'
import { ProjectModule } from './project/project.module'

@Module({
  imports: [
    forwardRef(() => AdminModule),  // ← match the child's forwardRef
    ProjectModule,                  // ← no forwardRef needed
    // ... other imports
  ],
  providers: [AuthService],
  exports: [AuthService, AdminModule, ProjectModule],
})
export class AuthModule {}
```

## Rules

1. Both sides of a circular dep must use `forwardRef`: `AdminModule` imports `forwardRef(() => AuthModule)` AND `AuthModule` imports `forwardRef(() => AdminModule)`.
2. Only one side needs `forwardRef` when the dependency is one-directional. `ProjectModule` doesn't need it because `ProjectGuard` doesn't inject `AuthService`.
3. `exports` must list the sub-modules so downstream consumers can import `AuthModule` once and get `AdminGuard`, `ProjectGuard`, etc.
