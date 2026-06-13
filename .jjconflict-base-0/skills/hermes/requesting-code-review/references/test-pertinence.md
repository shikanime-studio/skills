# Test pertinence and wording

Use this when reviewing or writing tests for a changed class.

## Heuristics

- Configuration derivation belongs in the configuration spec, not the consumer spec.
  - Example: `getKeycloakIssuer()` and `getKeycloakCertsUrl()` belong with `ConfigurationService`.
- Transport/fetch behavior belongs in the client wrapper spec.
  - Example: JWKS fetch, timeout, non-OK responses.
- Business rules and state reconciliation belong in the domain service spec.
  - Example: token validation, role merging, permission computation.
- Prefer `describe('<ClassName>')` and `it('should ...')` wording that states behavior, not internals.
- Avoid tests that duplicate helper logic or assert the wrong abstraction layer.

## Red flags

- A test names a consumer class but only verifies config string concatenation.
- A test suite mixes config derivation, network transport, and domain logic in the same file.
- A test title describes implementation mechanics instead of expected behavior.
