# Auth service boundaries

Session note: do not merge `AuthService` and `UserService` just because `TokenValidationResult` overlaps with `UserContext`.

## Boundary used in this repo
- `AuthService`: low-level token validation and permission resolution from persisted tokens.
- `UserService`: request-level adapter; reads headers, chooses the auth path, and returns the validated identity/permissions.
- `UserGuard`: thin guard; consumes `UserService` and writes the authenticated context onto the request.

## Rule of thumb
- Shared result shapes are fine.
- Shared responsibilities are not.
- If the only overlap is a DTO/result type, extract that type to a shared file instead of collapsing the services.

## When to consider merging
Only merge if both services have the same input boundary, the same callers, and no distinct lifecycle or persistence concerns. Otherwise keep the adapter/service split.
