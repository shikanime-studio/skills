---
name: sonarqube-refactoring
description:
  "Fix SonarQube quality gate issues: cognitive complexity, unnecessary
  assertions, typeof comparisons, duplicate imports, and code smells. Triggered
  when SonarQube reports are available, user mentions SonarQube issues, or PR
  checks fail on quality gates."
---

# SonarQube Refactoring

Fix code quality issues flagged by SonarQube analysis, especially for PR quality
gates.

## Common Issue Patterns & Fixes

### 1. Cognitive Complexity (S3776 — limit 15)

When a function exceeds 15 cognitive complexity, extract helper methods:

**Before:** One large method with nested `if`, `for`, `&&`, `||` **After:**
Extract 1-2 private methods per logical grouping

**Extraction targets:**

- Field-by-field update building → `applyXUpdates(tx, id, data)`
- Validation chains with early returns → `validateX(data, context)`
- Nested loops with conditionals → `buildXMap(data)`

**Target:** Reduce to ≤15. Aim for 2-3 points below threshold to leave headroom.

### 2. Unnecessary Type Assertions (S1905, S4325)

SonarQube flags `as X` when the assertion doesn't change the type.

**Pattern:** `getRequest() as { project?: ProjectContext }` where the return
type already includes the shape. **Fix:** Remove the `as` assertion entirely.

**Pattern:** `return projects as Project[]` where the function already declares
`Project[]` return type. **Fix:** Change the function signature to match the
actual data flow (accept/return `Partial<T>` if that's what flows through), or
remove the assertion if the types already align.

### 3. typeof undefined Comparison (S1535)

**Before:** `typeof rest.locked !== 'undefined'` **After:**
`rest.locked !== undefined`

### 4. as any on delete Operations (S4325)

**Before:**

```typescript
const effectiveData = { ...data }
delete (effectiveData as any).locked
if (project.locked && (effectiveData as any).locked !== false) { ... }
```

**After:**

```typescript
const effectiveData: Record<string, unknown> = { ...data }
delete effectiveData.locked
if (project.locked && effectiveData.locked !== false) { ... }
```

Using `Record<string, unknown>` makes `delete` and property access type-safe
without `as any`.

### 5. Duplicate Imports (S1128)

**Before:**

```typescript
import type { Prisma } from "@prisma/client";
import type { Project, ProjectMembers, User } from "@prisma/client";
```

**After:**

```typescript
import type { Prisma, Project, ProjectMembers, User } from "@prisma/client";
```

### 6. Replace if/else Chains with Zod (S3776 contributor)

When a function has 4+ `if/else if` branches doing type-based value conversion:

**Before:**

```typescript
function secretValueToString(value: unknown): string {
  if (typeof value === "string") return value;
  if (
    typeof value === "number" ||
    typeof value === "bigint" ||
    typeof value === "boolean"
  )
    return String(value);
  if (value === null || value === undefined) return "";
  return JSON.stringify(value);
}
```

**After:**

```typescript
const SecretValueSchema = z
  .union([
    z.string(),
    z.undefined().transform(() => ""),
    z.number().transform((v) => String(v)),
    z.bigint().transform((v) => String(v)),
    z.boolean().transform((v) => String(v)),
    z.null().transform(() => ""),
  ])
  .catch("");

// Usage:
groupObj[key] = SecretValueSchema.parse(value);
```

### 7. Replace Custom Concurrency Helpers with Promise.allSettled

**Before:** Custom `runWithConcurrency(tasks, limit)` with `Set<Promise>`,
`Promise.race`, manual tracking. **After:**
`Promise.allSettled(tasks.map(t => t()))` — simpler, standard, sufficient for
most cases.

## Procedure

1. **Read the SonarQube annotation list** — each issue has a rule ID and line
   number
2. **Group by type** — fix all instances of the same pattern together
3. **Fix simplest first** — `typeof` comparisons, duplicate imports, unnecessary
   `as`
4. **Then structural** — cognitive complexity via extraction, Zod replacement
5. **Run `tsc --noEmit`** after each group to verify no new type errors
6. **Run tests** — `pnpm test -- --run <affected-spec>` to verify behavior
   preserved

## Pitfalls

- **Extracting methods changes `this` context** — use arrow functions or
  `.bind(this)` when passing extracted methods as callbacks
- **`Record<string, unknown>` needs casts for Prisma** — when passing values
  from `Record<string, unknown>` to Prisma calls, cast at the boundary:
  `effectiveData.ownerId as string`
- **Zod `.catch('')` handles the fallback** — make sure the catch value matches
  the expected empty state for your domain
- **After extracting methods, update call sites** — the extracted method
  parameters must match what's available in the calling context
