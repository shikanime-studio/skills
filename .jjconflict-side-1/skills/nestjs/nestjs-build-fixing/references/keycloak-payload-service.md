# Keycloak payload schema inside the service

Session note: when refactoring Keycloak auth in `apps/server-nestjs`, the runtime Zod schema and inferred `KeycloakPayload` type were consolidated into `src/modules/infrastructure/auth/keycloak/keycloak.service.ts`.

Why this mattered:
- `makeKeycloakPayload()` in `keycloak-testing.utils.ts` now imports both `KeycloakPayloadSchema` and `KeycloakPayload` from the service boundary.
- `admin-permission.guard.ts` also imports `KeycloakPayloadSchema` from the service, so there is a single source of truth for the payload contract.
- The separate `keycloak-payload.schema.ts` file can be removed once all imports are updated.

Verification lesson:
- After moving the schema into the service, `KeycloakJwtService` unit tests failed until the spec provided a `PrismaService` mock in `Test.createTestingModule`.
- The failure mode was Nest DI resolution, not a TypeScript error.

Relevant paths:
- `apps/server-nestjs/src/modules/infrastructure/auth/keycloak/keycloak.service.ts`
- `apps/server-nestjs/src/modules/infrastructure/auth/keycloak/keycloak-testing.utils.ts`
- `apps/server-nestjs/src/modules/infrastructure/auth/keycloak/keycloak.service.spec.ts`
- `apps/server-nestjs/src/modules/infrastructure/auth/admin-permission.guard.ts`
