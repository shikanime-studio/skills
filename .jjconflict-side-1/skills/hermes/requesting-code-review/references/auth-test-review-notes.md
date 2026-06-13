# Auth and configuration test review notes

This session surfaced a few recurring test-quality patterns for the NestJS auth stack:

- Put URL/URI derivation tests in `ConfigurationService` specs when the value is a pure config getter.
- Keep `KeycloakJwtClientService` tests focused on HTTP fetch, timeout/abort handling, and response parsing.
- Keep `KeycloakJwtService` tests focused on cache lookups, key caching, and token-to-user/role reconciliation.
- Share guard `makeContext(...)` helpers under a common testing utility when multiple specs use the same Nest `ExecutionContext` setup.
- Prefer a mocked service instance (`mockDeep<T>()`) over ad hoc `vi.fn()` when a test is really about a dependency contract and not a single callback.
- Use class-style `describe('ClassName')` and intent-specific `it(...)` wording so the test file reads like a spec, not a log.
- If a helper belongs to configuration, move the getter to `ConfigurationService` and keep the consumer thin.
