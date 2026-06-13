# Repo Context

## Already Instrumented
- `modules/project/project.service.ts` — extensive `log`/`debug`/`error` on every method
- `modules/project-members/project-members.service.ts` — same coverage
- `modules/infrastructure/auth/auth.service.ts` — `Logger` with `debug`/`warn`/`error`
- `modules/infrastructure/auth/keycloak/keycloak.service.ts` — `Logger` with `log`/`error`

## Controllers Session Work
- `modules/project/project.controller.ts` — `Logger` imported and instance added; `getData` route logged
- `modules/project-members/project-members.controller.ts` — `Logger` imported and instance added; ready for route logs

## Pitfall
`modules/infrastructure/auth/auth.service.ts` contains a stray duplicate method declaration on `validateToken`. Inline patch during logging work can expose pre-existing syntax issues that differ from the pattern applied.
