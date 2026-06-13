---
name: test-driven-development
description: "TDD: enforce RED-GREEN-REFACTOR, tests before code."
version: 1.1.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [testing, tdd, development, quality, red-green-refactor]
    related_skills: [systematic-debugging, writing-plans, subagent-driven-development]
---

# Test-Driven Development (TDD)

## Overview

Write the test first. Watch it fail. Write minimal code to pass.

**Core principle:** If you didn't watch the test fail, you don't know if it tests the right thing.

**Violating the letter of the rules is violating the spirit of the rules.**

## When to Use

**Always:**
- New features
- Bug fixes
- Refactoring
- Behavior changes

**Exceptions (ask the user first):**
- Throwaway prototypes
- Generated code
- Configuration files

Thinking "skip TDD just this once"? Stop. That's rationalization.

## The Iron Law

```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
```

Write code before the test? Delete it. Start over.

**No exceptions:**
- Don't keep it as "reference"
- Don't "adapt" it while writing tests
- Don't look at it
- Delete means delete

Implement fresh from tests. Period.

## Red-Green-Refactor Cycle

### RED — Write Failing Test

Write one minimal test showing what should happen.

**Good test:**
```python
def test_retries_failed_operations_3_times():
    attempts = 0
    def operation():
        nonlocal attempts
        attempts += 1
        if attempts < 3:
            raise Exception('fail')
        return 'success'

    result = retry_operation(operation)

    assert result == 'success'
    assert attempts == 3
```
Clear name, tests real behavior, one thing.

**Bad test:**
```python
def test_retry_works():
    mock = MagicMock()
    mock.side_effect = [Exception(), Exception(), 'success']
    result = retry_operation(mock)
    assert result == 'success'  # What about retry count? Timing?
```
Vague name, tests mock not real code.

**Requirements:**
- One behavior per test
- Clear descriptive name ("and" in name? Split it)
- Real code, not mocks (unless truly unavoidable)
- Name describes behavior, not implementation

### Verify RED — Watch It Fail

**MANDATORY. Never skip.**

```bash
# Use terminal tool to run the specific test
pytest tests/test_feature.py::test_specific_behavior -v
```

Confirm:
- Test fails (not errors from typos)
- Failure message is expected
- Fails because the feature is missing

**Test passes immediately?** You're testing existing behavior. Fix the test.

**Test errors?** Fix the error, re-run until it fails correctly.

### GREEN — Minimal Code

Write the simplest code to pass the test. Nothing more.

**Good:**
```python
def add(a, b):
    return a + b  # Nothing extra
```

**Bad:**
```python
def add(a, b):
    result = a + b
    logging.info(f"Adding {a} + {b} = {result}")  # Extra!
    return result
```

Don't add features, refactor other code, or "improve" beyond the test.

**Cheating is OK in GREEN:**
- Hardcode return values
- Copy-paste
- Duplicate code
- Skip edge cases

We'll fix it in REFACTOR.

### Verify GREEN — Watch It Pass

**MANDATORY.**

```bash
# Run the specific test
pytest tests/test_feature.py::test_specific_behavior -v

# Then run ALL tests to check for regressions
pytest tests/ -q
```

Confirm:
- Test passes
- Other tests still pass
- Output pristine (no errors, warnings)

**Test fails?** Fix the code, not the test.

**Other tests fail?** Fix regressions now.

### REFACTOR — Clean Up

After green only:
- Remove duplication
- Improve names
- Extract helpers
- Simplify expressions

Keep tests green throughout. Don't add behavior.

**If tests fail during refactor:** Undo immediately. Take smaller steps.

### Repeat

Next failing test for next behavior. One cycle at a time.

## Why Order Matters

**"I'll write tests after to verify it works"**

Tests written after code pass immediately. Passing immediately proves nothing:
- Might test the wrong thing
- Might test implementation, not behavior
- Might miss edge cases you forgot
- You never saw it catch the bug

Test-first forces you to see the test fail, proving it actually tests something.

**"I already manually tested all the edge cases"**

Manual testing is ad-hoc. You think you tested everything but:
- No record of what you tested
- Can't re-run when code changes
- Easy to forget cases under pressure
- "It worked when I tried it" ≠ comprehensive

Automated tests are systematic. They run the same way every time.

**"Deleting X hours of work is wasteful"**

Sunk cost fallacy. The time is already gone. Your choice now:
- Delete and rewrite with TDD (high confidence)
- Keep it and add tests after (low confidence, likely bugs)

The "waste" is keeping code you can't trust.

**"TDD is dogmatic, being pragmatic means adapting"**

TDD IS pragmatic:
- Finds bugs before commit (faster than debugging after)
- Prevents regressions (tests catch breaks immediately)
- Documents behavior (tests show how to use code)
- Enables refactoring (change freely, tests catch breaks)

"Pragmatic" shortcuts = debugging in production = slower.

**"Tests after achieve the same goals — it's spirit not ritual"**

No. Tests-after answer "What does this do?" Tests-first answer "What should this do?"

Tests-after are biased by your implementation. You test what you built, not what's required. Tests-first force edge case discovery before implementing.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Too simple to test" | Simple code breaks. Test takes 30 seconds. |
| "I'll test after" | Tests passing immediately prove nothing. |
| "Tests after achieve same goals" | Tests-after = "what does this do?" Tests-first = "what should this do?" |
| "Already manually tested" | Ad-hoc ≠ systematic. No record, can't re-run. |
| "Deleting X hours is wasteful" | Sunk cost fallacy. Keeping unverified code is technical debt. |
| "Keep as reference, write tests first" | You'll adapt it. That's testing after. Delete means delete. |
| "Need to explore first" | Fine. Throw away exploration, start with TDD. |
| "Test hard = design unclear" | Listen to the test. Hard to test = hard to use. |
| "TDD will slow me down" | TDD faster than debugging. Pragmatic = test-first. |
| "Manual test faster" | Manual doesn't prove edge cases. You'll re-test every change. |
| "Existing code has no tests" | You're improving it. Add tests for the code you touch. |

## Red Flags — STOP and Start Over

If you catch yourself doing any of these, delete the code and restart with TDD:

- Code before test
- Test after implementation
- Test passes immediately on first run
- Can't explain why test failed
- Tests added "later"
- Rationalizing "just this once"
- "I already manually tested it"
- "Tests after achieve the same purpose"
- "Keep as reference" or "adapt existing code"
- "Already spent X hours, deleting is wasteful"
- "TDD is dogmatic, I'm being pragmatic"
- "This is different because..."

**All of these mean: Delete code. Start over with TDD.**

## Verification Checklist

Before marking work complete:

- [ ] Every new function/method has a test
- [ ] Watched each test fail before implementing
- [ ] Each test failed for expected reason (feature missing, not typo)
- [ ] Wrote minimal code to pass each test
- [ ] All tests pass
- [ ] Output pristine (no errors, warnings)
- [ ] Tests use real code (mocks only if unavoidable)
- [ ] Edge cases and errors covered

Can't check all boxes? You skipped TDD. Start over.

## When Stuck

| Problem | Solution |
|---------|----------|
| Don't know how to test | Write the wished-for API. Write the assertion first. Ask the user. |
| Test too complicated | Design too complicated. Simplify the interface. |
| Must mock everything | Code too coupled. Use dependency injection. |
| Test setup huge | Extract helpers. Still complex? Simplify the design. |

## Hermes Agent Integration

### Running Tests

Use the `terminal` tool to run tests at each step:

```python
# RED — verify failure
terminal("pytest tests/test_feature.py::test_name -v")

# GREEN — verify pass
terminal("pytest tests/test_feature.py::test_name -v")

# Full suite — verify no regressions
terminal("pytest tests/ -q")
```

### With Vitest + Playwright (Dual Test Strategy)

For full-stack React projects, use the dual test strategy:

1. **Vitest** for unit tests — data model validation, Zod schemas, route structure, TanStack React DB collections. Fast (< 1s).
2. **Playwright** for E2E — navigation, form interaction, responsive layout, keyboard shortcuts. Runs against dev server.

See `references/vitest-playwright-strategy.md` for setup config, workflow, and pitfalls.

```python
# RED — unit test first
terminal("npx vitest run --reporter=verbose")

# GREEN — implement, re-run unit
terminal("npx vitest run")

# E2E test against dev server
terminal("npx playwright test", timeout=120000, workdir="apps/reiya")
```

### With delegate_task

When dispatching subagents for implementation, enforce TDD in the goal:

```python
delegate_task(
    goal="Implement [feature] using strict TDD",
    context="""
    Follow test-driven-development skill:
    1. Write failing test FIRST
    2. Run test to verify it fails
    3. Write minimal code to pass
    4. Run test to verify it passes
    5. Refactor if needed
    6. Commit

    Project test command: pytest tests/ -q
    Project structure: [describe relevant files]
    """,
    toolsets=['terminal', 'file']
)
```

### With systematic-debugging

Bug found? Write failing test reproducing it. Follow TDD cycle. The test proves the fix and prevents regression.

Never fix bugs without a test.

## Testing Anti-Patterns

- **Testing mock behavior instead of real behavior** — mocks should verify interactions, not replace the system under test
- **Testing implementation details** — test behavior/results, not internal method calls
- **Happy path only** — always test edge cases, errors, and boundaries
- **Brittle tests** — tests should verify behavior, not structure; refactoring shouldn't break them

## NestJS + Vitest Testing Patterns

When writing unit tests for NestJS services with Vitest:

- **No decorative section comments** — no `// === methodName ===` or `// --- Helpers ---` dividers in spec files. `describe`/`it` blocks provide enough structure.
- **Keep Nest testing module setup local to the spec** when matching the GitLab/NestJS style used in this repo; `*-testing.utils.ts` should focus on data factories and reusable mocks.
- **Use `vitest-mock-extended`'s `mockDeep<T>()`** for Prisma/DI service mocks — not `Partial<T>` with `vi.fn()`, which loses `.mockResolvedValue()` type safety under NestJS DI.
- **Use type-shaped `make<TypeName>` factory names** in testing utilities (`makeProject`, `makeCreateProjectBody`, `makeProjectWithDetails`, `makeTxMock`). Remove pass-through helpers that only return their input; inline simple request bodies in specs.
- **Mock Prisma transactions directly** with `prisma.$transaction.mockImplementation(async cb => cb(tx as ...))`; do not add a separate `txFn` indirection unless the existing spec already standardizes on it.
- **Use `faker`** for test data with factory functions accepting `overrides` spread last.

See `references/nestjs-vitest-testing-patterns.md` for detailed patterns including mocking, factory functions, transaction testing, typed mock result helpers, the `as never` anti-pattern, and the `mockDeep`-returns-`undefined` pitfall.

## Pitfall: `as never` on `mockResolvedValue` — Use Typed Mock Result Helpers

`as never` on `.mockResolvedValue()` disables all type checking — real type errors in test fixtures become silent.

**Best fix:** add typed mock result helpers in `*-testing.utils.ts` so the spec file stays completely free of inline `as` casts:

```typescript
// In *-testing.utils.ts
export function makeProjectFindManyResult(projects = []): Project[] {
  return projects as Project[]
}
export function makeProjectCreateResult(overrides: Partial<Project> & { id: string }): Project {
  return overrides as Project
}
export function makeProjectWithMembersResult(project, members = []): ProjectWithMembers {
  return { ...project, members } as ProjectWithMembers
}
```

Then in the spec: `tx.project.create.mockResolvedValue(makeProjectCreateResult({ id: pwd.id }))` — no inline `as` at all.

**For one-off cases:** use `as Project` / `as Project[]` inline (never `as never`). Use `satisfies Pick<Project, 'field'>` on partial objects to catch typos at compile time.

See `references/nestjs-vitest-testing-patterns.md` for the full "Typed Mock Result Helpers" pattern.

## Pitfall: `mockDeep` Silently Returns `undefined`

When using `vitest-mock-extended`'s `mockDeep<T>()`, any nested method you don't explicitly configure returns `undefined` (not a promise). This causes `TypeError: Cannot read properties of undefined (reading '...')` at the **service code line**, not in the test — making it hard to diagnose.

**Fix**: Audit every `tx.xxx.yyy` (or `prisma.xxx.yyy`) call in the code path under test and mock each one, even with minimal values like `mockResolvedValue([])` or `mockResolvedValue({ id: '...' })`.

### NestJS Dependency Resolution
When moving methods between services (e.g., during migration like moving `validateKeycloakPayload` from `AuthService` to `KeycloakJwtService`), you may introduce new dependencies (such as `PrismaService`, `ConfigurationService`, `CACHE_MANAGER`). This causes test failures with `Error: Nest can't resolve dependencies of the [ServiceName] (...)`.

**Fix protocol:**
1. **Verify dependencies**: Check the target service's constructor for new dependencies.
2. **Update test providers**: Add the new dependency to the `providers` array in `Test.createTestingModule`.
3. **Mock the entire chain**: Pre-configure every nested method the service uses (even if returning `null`/empty arrays).
4. **Watch for circular dependencies**: Ensure the mock setup doesn't create circular reference issues.

Example for migrating a method to `KeycloakJwtService`:
```typescript
// In keycloak.service.spec.ts
beforeEach(async () => {
  prisma = mockDeep<PrismaService>()
  config = mockDeep<ConfigurationService>()
  cache = mockDeep<Cache>()
  
  // CRITICAL: Mock all nested calls the service makes
  prisma.user.findUnique.mockResolvedValue(null)
  prisma.adminRole.findMany.mockResolvedValue([])
  // ... other nested calls
  
  module = await Test.createTestingModule({
    providers: [
      KeycloakJwtService,
      { provide: PrismaService, useValue: prisma },
      { provide: ConfigurationService, useValue: config },
      { provide: CACHE_MANAGER, useValue: cache },
    ],
  }).compile()
})
```

## Final Rule

```
Production code → test exists and failed first
Otherwise → not TDD
```

No exceptions without the user's explicit permission.
