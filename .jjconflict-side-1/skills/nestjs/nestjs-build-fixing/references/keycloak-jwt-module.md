# Keycloak JWT Module — File Layout

Standard file layout for the `auth/keycloak/` subfolder when migrating JWT validation to NestJS:

```
auth/
├── keycloak/
│   ├── keycloak.service.ts          # KeycloakJwtService: JWKS caching, header decoding
│   ├── keycloak.module.ts           # KeycloakJwtModule: provides KeycloakJwtService
│   ├── keycloak.service.spec.ts     # Unit tests for decodeJweHeader, getPublicKey
│   ├── keycloak-payload.schema.ts   # Zod schema for JWT payload validation
│   └── keycloak-testing.utils.ts    # makeKeycloakPayload(overrides?) factory
├── auth.module.ts                   # AuthModule: imports KeycloakJwtModule + JwtModule
├── auth.service.ts                  # AuthService: validateToken, validateKeycloakPayload
├── auth.service.spec.ts             # Unit tests for token + JWT payload validation
├── admin-permission.guard.ts        # Guard: x-dso-token OR Bearer JWT auth
├── admin-permission.guard.spec.ts   # Guard tests
└── admin-permission.decorator.ts    # @RequireAdminPermission()
```

## Key Design Decisions

- **`decodeJweHeader` accepts `string | object | Buffer<ArrayBufferLike>`**: Matches the `secretOrKeyProvider` callback signature from `@nestjs/jwt`. Returns `null` for non-string input.
- **`getPublicKey` returns `string | undefined`**: PEM-encoded public key string (not `KeyObject`).
- **`getIssuer()` is a method**: Builds the issuer URL from config, reused in both `JwtModule` setup and tests.
- **No `validateJwt` method on `KeycloakJwtService`**: The guard calls `jwtService.verifyAsync()` directly through `@nestjs/jwt`. `KeycloakJwtService` only handles JWKS caching and header decoding.
- **Separate `KeycloakJwtModule`**: Avoids circular dependency between `AuthModule` and `KeycloakJwtService`.

## Zod Payload Schema Pattern

```typescript
// keycloak-payload.schema.ts
import { z } from 'zod'

const coerceToString = (val: unknown) => (typeof val === 'string' ? val : '')
const coerceToStringArray = (val: unknown) =>
  Array.isArray(val) ? val.filter((v: unknown) => typeof v === 'string') : []

export const KeycloakPayloadSchema = z.object({
  sub: z.string(),
  email: z.preprocess(coerceToString, z.string().default('')),
  given_name: z.preprocess(coerceToString, z.string().default('')),
  family_name: z.preprocess(coerceToString, z.string().default('')),
  groups: z.preprocess(coerceToStringArray, z.array(z.string()).default([])),
})

export type KeycloakPayload = z.infer<typeof KeycloakPayloadSchema>
```

Usage in guard:
```typescript
const rawPayload = await this.jwtService.verifyAsync(jwt)
const payload = KeycloakPayloadSchema.parse(rawPayload)
```

## Faker Test Factory Pattern

```typescript
// keycloak-testing.utils.ts
import { z } from 'zod'
import { faker } from '@faker-js/faker'
import { KeycloakPayloadSchema } from './keycloak-payload.schema'

export function makeKeycloakPayload(
  overrides: Partial<z.infer<typeof KeycloakPayloadSchema>> = {},
) {
  return KeycloakPayloadSchema.parse({
    sub: faker.string.uuid(),
    email: faker.internet.email().toLowerCase(),
    given_name: faker.person.firstName(),
    family_name: faker.person.lastName(),
    groups: Array.from({ length: faker.number.int({ min: 0, max: 3 }) }, () =>
      `/${faker.word.noun()}`,
    ),
    ...overrides,
  })
}
```

Follows the project's existing `make*` factory pattern (see `keycloak/keycloack-testing.utils.ts`).
