# Plugin logger implementation

Reference for adding structured logging to console plugins.

## Files changed

1. `src/logger.ts` — plugin child logger
2. `package.json` — add `@cpn-console/logger` dependency
3. `src/functions.ts` — import and instrument exported step calls

## Example files

- `plugins/vault/src/logger.ts`
- `plugins/sonarqube/src/logger.ts`
- `plugins/argocd/src/logger.ts`

## Log level policy

- `info` — hook enter and successful completion in exported step functions
- `debug` — intermediate state inside helpers (`removeInfraEnvValues`, `cleanupProjectInfra`, environment-loop milestones)
- `error` — caught exceptions; include structured fields `action`, identifier, `err`

## Structured fields

Use `projectSlug` when available. For resources without slugs, use the corresponding object ID or name. Always log `action` matching the exported function name.
