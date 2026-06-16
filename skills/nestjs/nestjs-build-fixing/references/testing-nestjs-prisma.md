# Testing NestJS Services with Prisma — Mocking Patterns

## The Correct Pattern: `vitest-mock-extended`

Use `mockDeep<PrismaService>()` from `vitest-mock-extended` for all Prisma
mocking in NestJS unit tests.

## What NOT to Do

### Don't use `as never` on `mockResolvedValue` calls

```typescript
// WRONG — bottom type escape hatch, zero type safety
tx.project.findMany.mockResolvedValue([] as never);
tx.project.create.mockResolvedValue({ id: "test-id" } as never);
tx.project.findUniqueOrThrow.mockResolvedValue({
  ...pwd,
  members: [],
} as never);
```

## Replace `as never` with Typed Factory Functions

Extract mock return values into typed factory functions in `*-testing.utils.ts`:

```typescript
import type { Prisma } from "@prisma/client";
import type { Project } from "@prisma/client";

export type ProjectWithMembers = Prisma.ProjectGetPayload<{
  include: { members: { include: { user: true } } };
}>;

export function makeProjectFindManyResult(
  projects: Array<Partial<Project>> = [],
): Project[] {
  return projects as Project[];
}

export function makeProjectCreateResult(overrides: Partial<Project>): Project {
  return overrides as Project;
}

export function makeProjectWithMembersResult(
  project: ProjectWithDetails,
  members: Array<
    Prisma.ProjectMembersGetPayload<{ include: { user: true } }>
  > = [],
): ProjectWithMembers {
  return { ...project, members } as ProjectWithMembers;
}
```

Usage in spec:

```typescript
// Before (unsafe):
tx.project.findMany.mockResolvedValue([] as never);

// After (typed factory):
tx.project.findMany.mockResolvedValue(makeProjectFindManyResult());
```

### Key rules for factory functions

- Accept `Partial<Project>` not `Partial<Project> & { id: string }` — mocks are
  intentionally partial
- Return the full Prisma model type — the `as` cast is encapsulated in the
  factory
- For `findUniqueOrThrow` with includes, use
  `Prisma.ProjectGetPayload<{ include: { ... } }>` as the return type
- Keep factories in `*-testing.utils.ts` alongside other `make*` helpers

## Transaction Client Mock Pattern

```ts
const tx = mockDeep<Prisma.TransactionClient>();
tx.project.findMany.mockResolvedValue(makeProjectFindManyResult());
prisma.$transaction.mockImplementation(async (cb) => cb(tx));
```

## Common Gotchas

- **Re-mock per test**: Set `.mockResolvedValue()` inside each `it()`, not
  `beforeEach`
- **Type assertion**: Cast with `as unknown as DeepMockProxy<PrismaService>`
- **`findUnique` vs `findUniqueOrThrow`**: Separate mock methods — set returns
  on the correct one
- **Prisma mock shapes must include ALL required fields**:
  `mockDeep<PrismaService>()` enforces strict return types. For
  `PersonalAccessToken`, required fields include `id`, `name`, `status`,
  `expirationDate` (non-nullable `DateTime`), `lastUse`, `createdAt`, `hash`,
  `userId`, and the `owner` relation (when `include: { owner: true }` is used).
  For `AdminToken`, `expirationDate` is nullable (`DateTime?`) but
  `PersonalAccessToken.expirationDate` is NOT nullable. Always check the Prisma
  schema for nullability before setting mock values. A common error is
  `Type 'null' is not assignable to type 'Date'` when passing
  `expirationDate: null` to a non-nullable field.
- **Mocking nested relations**: When the code uses `include: { owner: true }`,
  the mock must include the `owner` relation object with at least `id` and
  `adminRoleIds` fields. The full shape depends on what the business logic
  accesses.
