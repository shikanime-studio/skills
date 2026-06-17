# Auth / JWT review notes

Use this when reviewing auth changes that introduce JWT verification or JWKS
lookup.

## Blocking checks

- Ensure tokens are bound to the intended client, not just the realm.
  - Look for `aud`, `azp`, or explicit client-id checks.
- Ensure JWKS refreshes are not unbounded on the request path.
  - Prefer a configurable timeout and abort handling.
- Ensure cache-miss behavior is safe.
  - A missing key should fail closed, not silently accept an unverified token.
- Ensure issuer and allowed algorithms are constrained.

## Verification pattern

- Run focused auth tests first.
- Add or update a test for:
  - audience/client binding
  - JWKS cache miss + refresh
  - fetch timeout / abort path

## Useful implementation details

- In Node 18+, `fetch(url, { signal: AbortSignal })` can be aborted with
  `AbortController`.
- Keep JWKS TTL and JWKS fetch timeout as separate configuration values.
- If a JWT path needs Keycloak group claims, confirm the realm exports `groups`
  in the token.
