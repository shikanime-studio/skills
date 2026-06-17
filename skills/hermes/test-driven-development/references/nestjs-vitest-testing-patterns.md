# NestJS + Vitest Testing Patterns

<!-- markdownlint-disable MD013 -->

## Test File Structure

**No decorative section comments** in spec files — no `// === methodName ===` or
`// --- Helpers ---` dividers. `describe`/`it` blocks provide enough structure.

**Keep Nest testing module setup local to the spec** when following the
GitLab-style pattern used in this repo. Use `*-testing.utils.ts` for reusable
data factories and mocks only; avoid exporting a generic
`make<ProjectService>TestingModule()` unless the existing module already
standardizes on that.

```text
project.service.spec.ts          ← test cases + local
  createProjectServiceTestingModule()
project-testing.utils.ts         ← factories and reusable mock objects only
```

Import in the spec:

```typescript
import {
  makeCreateProjectBody,
  makeProject,
  makeProjectContext,
  makeProjectMembers,
  makeProjectWithDetails,
  makeTxMock,
  makeUser,
} from "./project-testing.utils";
```

Testing utils file exports:

- `make<TypeName>()` — faker-based factory functions for each real/domain test
  data shape (`makeProject`, `makeUser`, `makeCreateProjectBody`,
  `makeProjectWithDetails`, etc.).
- `makeTxMock()` — shared transaction callback mock with overrides for the
  Prisma delegates used by the service.
- Typed mock result helpers — `makeProjectFindManyResult`,
  `makeProjectCreateResult`, `makeProjectWithMembersResult` — encapsulate the
  `as` casts needed for `mockDeep` strict generics (see "Typed Mock Result
  Helpers" below).

Avoid exporting pass-through body builders such as
`makeUpdateProjectBody(overrides) => overrides`, `makeAddMemberBody`,
`makePatchMembersBody`, or `makeBulkActionBody`. Inline simple request objects
in specs; only keep factories that construct a real typed object with defaults.

## Mocking Prisma with vitest-mock-extended

**Always use `mockDeep<T>()`** from `vitest-mock-extended` for services injected
via NestJS DI (PrismaService, etc.):

```typescript
import { mockDeep } from "vitest-mock-extended";
import { PrismaService } from "../infrastructure/database/prisma.service";

function createTestingModule() {
  const prisma = mockDeep<PrismaService>();
  return {
    module: Test.createTestingModule({
      providers: [MyService, { provide: PrismaService, useValue: prisma }],
    }),
    prisma,
  };
}
```

In `beforeEach`, cast the resolved service:

```typescript
let prisma: DeepMockProxy<PrismaService>;

beforeEach(async () => {
  const { module } = createProjectServiceTestingModule();
  const moduleRef = await module.compile();
  prisma = moduleRef.get(
    PrismaService,
  ) as unknown as DeepMockProxy<PrismaService>;
});
```

**Why not `Partial<PrismaService>`?** NestJS DI infers property types from the
`useValue` shape. A plain object `{ findMany: vi.fn() }` is typed as the real
Prisma delegate (e.g. `ProjectDelegate`), not a mock. Calling
`.mockResolvedValue()` on it fails type-checking. `mockDeep<T>()` generates a
complete recursive mock that satisfies the full type.

## Factory Functions Pattern

Each factory creates a valid test data object with sensible defaults and
`overrides` spread last:

```typescript
export const makeProjectWithDetails = (
  overrides: Partial<ProjectWithDetails> = {},
): ProjectWithDetails => {
  const owner = makeOwner();
  return {
    id: faker.string.uuid(),
    name: faker.string.alphanumeric(8).toLowerCase(),
    slug: faker.string.alphanumeric(8).toLowerCase(),
    description: faker.lorem.sentence(),
    status: "created",
    locked: false,
    limitless: false,
    hprodCpu: 1,
    hprodGpu: 0,
    hprodMemory: 2,
    prodCpu: 1,
    prodGpu: 0,
    prodMemory: 2,
    everyonePerms: PROJECT_PERMS.GUEST,
    ownerId: owner.id,
    createdAt: faker.date.recent(),
    updatedAt: faker.date.recent(),
    lastSuccessProvisionningVersion: null,
    owner,
    members: [],
    plugins: [],
    roles: [],
    repositories: [],
    environments: [],
    deployments: [],
    clusters: [],
    ...overrides,
  };
};
```

Key rules:

- Name exported helpers as `make<TypeName>()`, matching the object or testing
  artifact they return.
- Prefer type-shaped names (`makeCreateProjectBody`, `makeProjectWithDetails`,
  `makeTxMock`) over role/row/action names (`makeOwner`, `makeProjectRow`,
  `makeAddMemberBody`). A `makeXxxWithYyy` name is acceptable only when it
  matches a real exported domain type such as `ProjectWithDetails`; otherwise
  split or rename it.
- Always include `faker.string.uuid()` for IDs that are UUIDs
- Always include all array fields as `[]` (Prisma select returns arrays)
- Always include all required fields — use explicit return type annotation and
  `satisfies` to catch misses
- Spread `overrides` **last** so tests can replace any field
- Keep factories minimal — only add what tests actually need
- Delete useless factories that merely return their input; inline those literals
  in the spec.
- Return full Prisma-shaped objects from mocked Prisma calls; avoid `{}` or
  partial `{ id }` values unless the mocked method's return type is explicitly
  narrowed.

## Typed Mock Result Helpers

When `mockDeep<Prisma.TransactionClient>()` creates strict generics, partial
test fixtures don't match the exact Prisma return type. Rather than scattering
`as Project` / `as Project[]` / `as Prisma.ProjectGetPayload<...>` inline
throughout the spec (ugly, repetitive, easy to get wrong), encapsulate each cast
in a typed helper function in `*-testing.utils.ts`:

```typescript
// In project-testing.utils.ts

import type { Prisma } from "@prisma/client";
import type { Project } from "@prisma/client";

/** Shape returned by Prisma `tx.project.findUniqueOrThrow` with members
  include. */
export type ProjectWithMembers = Prisma.ProjectGetPayload<{
  include: { members: { include: { user: true } } };
}>;

/** Create a mock result for `tx.project.findMany` / `prisma.project.findMany`.
 */
export function makeProjectFindManyResult(
  projects: Array<Partial<Project> & { id: string }> = [],
): Project[] {
  return projects as Project[];
}

/** Create a mock result for `tx.project.create` (or any single-record create).
 */
export function makeProjectCreateResult(
  overrides: Partial<Project> & { id: string },
): Project {
  return overrides as Project;
}

/** Create a mock result for `tx.project.findUniqueOrThrow` with members
  include. */
export function makeProjectWithMembersResult(
  project: ProjectWithDetails,
  members: Array<
    Prisma.ProjectMembersGetPayload<{ include: { user: true } }>
  > = [],
): ProjectWithMembers {
  return { ...project, members } as ProjectWithMembers;
}
```

**Usage in spec — clean, no inline `as` casts:**

```typescript
// Before (inline casts scattered through spec):
tx.project.findMany.mockResolvedValue(existingSlugs.map(slug => ({ slug })) as
  never)
tx.project.create.mockResolvedValue({ id: pwd.id } as never)
tx.project.findUniqueOrThrow.mockResolvedValue({ ...pwd, members: [] } as
  Prisma.ProjectGetPayload<...>)

// After (typed helpers in testing utils, spec is cast-free):
tx.project.findMany.mockResolvedValue(makeProjectFindManyResult(existingSlugs.map(slug => ({ slug }))))
tx.project.create.mockResolvedValue(makeProjectCreateResult({ id: pwd.id }))
tx.project.findUniqueOrThrow.mockResolvedValue(makeProjectWithMembersResult(pwd))
tx.project.findUniqueOrThrow.mockResolvedValue(
  makeProjectWithMembersResult(pwd, [makeProjectMemberWithUser(newUser, {
    roleIds: ['r1'] })]),
)
```

**Rules for the `as` cast inside helpers:**

- Use `as Project` / `as Project[]` for simple model returns (not `as never` —
  still type-checked)
- Use `Prisma.ProjectGetPayload<{ include: ... }>` matching the actual service
  `include` for relation-loaded returns
- Never use `as any` or `as never` — if the Prisma return type is complex, add a
  dedicated helper with the correct `ProjectGetPayload` type
- The helpers belong in the `*-testing.utils.ts` file so the spec file stays
  completely free of inline `as` assertions

## Pitfall: `as never` on `mockResolvedValue` — Use Proper Types or Helpers

**Never use `as never` on `.mockResolvedValue()`.** It disables all type
checking on the mock return, hiding real mistakes (missing fields, typos, wrong
shape).

**Resolution order (pick the first that fits):**

1. **Add a typed mock result helper** in `*-testing.utils.ts` (see "Typed Mock
   Result Helpers" above) — best for patterns used 3+ times
2. **Inline `as Project`** as a last resort for truly one-off casts — never
   `as never`

```typescript
// BAD — silences ALL type checking
tx.project.create.mockResolvedValue({ id: pwd.id } as never);

// ACCEPTABLE — one-off inline cast if the pattern is truly unique
tx.project.create.mockResolvedValue({ id: pwd.id } as Project);

// BEST — typed helper in testing utils (spec stays cast-free)
tx.project.create.mockResolvedValue(makeProjectCreateResult({ id: pwd.id }));
```

## Pitfall: `mockDeep` Returns `undefined` for Unmocked Methods

`mockDeep<T>()` from `vitest-mock-extended` creates a recursive mock proxy, but
any method you don't explicitly configure with `.mockResolvedValue()` (or
similar) returns `undefined` — **not** a no-op promise. This causes
`TypeError: Cannot read properties of undefined` at the call site, not at the
mock declaration.

Common failure pattern when testing Prisma transaction callbacks:

```typescript
const tx = mockDeep<Prisma.TransactionClient>();
tx.project.findUnique.mockResolvedValue(someValue);
prisma.$transaction.mockImplementation(async (cb) => cb(tx));
// tx.project.create is NOT mocked → returns undefined
// service code: const created = await tx.project.create(...)
//               created.id  → TypeError: Cannot read properties of undefined
  (reading 'id')
```

**The error message references the SERVICE code line, not the test.** Always
audit the service method under test and ensure EVERY `tx.xxx.yyy` call that
executes has a corresponding mock.

**Debugging technique**: When the error is cryptic
(`Cannot read properties of undefined (reading 'map')` or `...'id'`), look at
the stack trace line number in the service file. Identify what's being accessed
(`.id`, `.map`, `.slug`), then trace backward to find which mock was missing.

**Systematic fix approach**:

1. Read the service method completely
2. Identify every `tx.` call inside the transaction callback
3. Mock each one — even with a minimal value
4. Use `mockResolvedValue([])` for `findMany` even if the test doesn't assert on
   it
5. Use `mockResolvedValue({ id: '...' })` for `create` even if the test doesn't
   assert on the create args

Same applies to non-transaction mocks: if the service calls
`prisma.project.findMany()` (outside a transaction), that must be mocked too.

## Transaction Mocking

For methods using `prisma.$transaction()`, mock the callback pattern through a
reusable factory in `*-testing.utils.ts`:

```typescript
export interface TxMock {
  project: {
    findUniqueOrThrow: ReturnType<typeof vi.fn>;
    update: ReturnType<typeof vi.fn>;
    findUnique: ReturnType<typeof vi.fn>;
    findMany: ReturnType<typeof vi.fn>;
    create: ReturnType<typeof vi.fn>;
  };
  projectMembers: {
    create: ReturnType<typeof vi.fn>;
    delete: ReturnType<typeof vi.fn>;
    upsert: ReturnType<typeof vi.fn>;
  };
}

export function makeTxMock(
  overrides: {
    project?: Partial<TxMock["project"]>;
    projectMembers?: Partial<TxMock["projectMembers"]>;
  } = {},
): TxMock {
  return {
    project: {
      findUniqueOrThrow: vi
        .fn()
        .mockResolvedValue({ ...makeProjectWithDetails(), members: [] }),
      update: vi.fn().mockResolvedValue(makeProject()),
      findUnique: vi.fn(),
      findMany: vi.fn().mockResolvedValue([]),
      create: vi.fn().mockResolvedValue({ id: faker.string.uuid() }),
      ...overrides.project,
    },
    projectMembers: {
      create: vi.fn(),
      delete: vi.fn().mockResolvedValue(makeProjectMembers()),
      upsert: vi.fn().mockResolvedValue(makeProjectMembers()),
      ...overrides.projectMembers,
    },
  };
}
```

In tests, mock `$transaction` directly:

```typescript
import type { Prisma } from "@prisma/client";

const tx = makeTxMock({
  project: {
    findUniqueOrThrow: vi.fn().mockResolvedValue({ ...pwd, members: [] }),
  },
});

prisma.$transaction.mockImplementation(async (cb) =>
  cb(tx as unknown as Prisma.TransactionClient),
);
```

Then assert on `tx.project.update` etc. — not the outer `prisma` mock. Avoid a
separate `txFn` wrapper unless an existing spec already uses that abstraction
consistently.

## Test Coverage Checklist

For each service method, test at minimum:

- Happy path (success + return value shape)
- Not found → `NotFoundException`
- Forbidden / permission denied → `ForbiddenException`
- Validation errors → `BadRequestException` / `UnprocessableEntityException`
- Edge cases (empty arrays, null fields, boundary values)
