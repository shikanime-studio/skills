# Testing utility factory style for `apps/server-nestjs`

Session-derived pattern from refactoring `project-testing.utils.ts`.

## Preferred shape

- Keep `*-testing.utils.ts` focused on reusable data factories and mock/data builders.
- Use `make<TypeName>(overrides?: Partial<TypeName>): TypeName` where `<TypeName>` maps to an actual model, DTO, query, context, or named project type.
- Keep Nest `Test.createTestingModule(...)` helpers local to the relevant spec file.
- Name local module builders `create<Subject>TestingModule()`.

## Prisma Mock Return Factories

When a spec has many `mockResolvedValue(... as never)` calls for Prisma transaction client methods, extract typed factory functions:

```typescript
// In *-testing.utils.ts
export type ProjectWithMembers = Prisma.ProjectGetPayload<{
  include: { members: { include: { user: true } } }
}>

export function makeProjectFindManyResult(
  projects: Array<Partial<Project>> = [],
): Project[] {
  return projects as Project[]
}

export function makeProjectCreateResult(
  overrides: Partial<Project>,
): Project {
  return overrides as Project
}

export function makeProjectWithMembersResult(
  project: ProjectWithDetails,
  members: Array<Prisma.ProjectMembersGetPayload<{ include: { user: true } }>> = [],
): ProjectWithMembers {
  return { ...project, members } as ProjectWithMembers
}
```

This replaces inline `as never` casts in specs with properly-typed factory calls. The factory encapsulates the `as` cast and returns the correct Prisma type.

### Rules

- Accept `Partial<Project>` — mocks are intentionally partial, don't require fields the test doesn't use
- Return the full Prisma model type — compiler verifies the mock matches the expected shape
- For `findUniqueOrThrow` with includes, return `Prisma.ProjectGetPayload<{ include: { ... } }>`
- All inline `as` casts on `mockResolvedValue` should be eliminated from spec files

## Avoid

- Do not export ad hoc relation helpers when Prisma `include` shapes can be composed inline.
- Do not use `as never` on `mockResolvedValue` calls — use typed factories instead.
- Do not require `id` in mock factory parameters when the test only needs a subset of fields.
- Do not use inline `vi.fn()` with `as any` for Prisma delegates — use `mockDeep`.
- Do not use `Partial<PrismaService>` with `vi.fn()` — use `mockDeep<PrismaService>()`.
