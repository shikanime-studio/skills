# Keycloak JWT review notes

Use these checks when reviewing NestJS auth changes that add Keycloak JWT support.

- Verify tokens are bound to the backend client, not only the realm.
  - `issuer` alone is not enough for service auth.
  - Check `aud` and/or `azp` against the configured client id.
- Keep JWKS retrieval off the hot path where possible.
  - If the code fetches JWKS on cache miss, require a timeout and a clear failure mode.
  - Prefer a dedicated client/fetch service over inline network calls inside auth policy code.
- Cache materialized public keys by `kid`.
  - Refresh on unknown `kid`.
  - Do not assume the cache is populated; verify the miss path.
- In tests, mock the client/fetch layer separately from the authorization logic.
  - This keeps `validatePayload`/permission logic deterministic.
  - Add coverage for cache hit, cache miss, fetch failure, and timeout/abort.
