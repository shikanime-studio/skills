# Keycloak JWT auth review notes

Session lesson:
- When adding a new bearer JWT path for Keycloak, verify both issuer and client binding.
- Realm issuer alone is not enough; check `aud` (or equivalent client binding such as `azp`) against the backend client id.
- JWKS retrieval should have a timeout/abort path so a slow identity provider does not stall request handling.

Verified locally during this session:
- `pnpm test -- src/modules/infrastructure/auth/auth.service.spec.ts src/modules/infrastructure/auth/user.guard.spec.ts src/modules/infrastructure/auth/keycloak-jwt/keycloak-jwt.service.spec.ts src/modules/infrastructure/auth/admin-permission.guard.spec.ts`
- Result: 28 files passed, 202 tests passed, 18 skipped.
