---
name: nestjs-logging
description: Add consistent, debug-friendly logging to NestJS modules in this repo, using the existing LoggerModule + @nestjs/common Logger pattern.
---

# NestJS Logging Instrumentation

Use when adding or reviewing logging in `apps/server-nestjs/src/**` controllers, services, guards, or utilities.

## Prerequisites / Context

- `LoggerModule` in `apps/server-nestjs/src/modules/infrastructure/logger/logger.module.ts` already configures `nestjs-pino` with redaction and environment-specific transports.
- No additional imports are needed beyond `@nestjs/common`.

## Pattern

### Classes
```ts
import { Logger } from '@nestjs/common'

class SomeService {
  private readonly logger = new Logger(SomeService.name)

  doWork(input: string) {
    this.logger.debug(`someService.doWork started (input=${input})`)
    try {
      // ...
      this.logger.log('someService.doWork completed')
    } catch (error) {
      this.logger.error(
        `someService.doWork failed: ${error instanceof Error ? error.message : String(error)}`,
        error instanceof Error ? error.stack : undefined,
      )
      throw error
    }
  }
}
```

### Controllers
Log only where it improves debuggability:
```ts
class SomeController {
  private readonly logger = new Logger(SomeController.name)

  @Get('')
  async list() {
    return (await this.service.list()).map(format)
  }
}

@Post('')
@HttpCode(201)
async create(@Body() body, @User() user) {
  this.logger.log(`someController.create (userId=${user.id}, name=${body.name})`)
  return this.service.create(body, user.id)
}
```

## Levels / Conventions

- `debug` — routine entry/exit, counts, non-sensitive metadata.
- `log` — significant business events (project.create, project.update ownerChange).
- `warn` — suspicious but non-error states (missing token, forbidden filter).
- `error` — caught exceptions; message + stack.

## Bulk Edits

When instrumenting multiple files with the same pattern, prefer one bulk `write_file` or `patch` over many small edits. Keep message style consistent: `className.methodName event (key=value, ...)`.

## Pitfalls / User Preferences

- Never use `console.log`; use `Logger`.
- Keep log messages parseable: include IDs (`projectId`, `userId`), counts, and short action names.
- `Logger` constructor takes a string; use `ClassName.name`, not a literal.
- Do not wrap controller methods in `try/catch` solely to log and rethrow; NestJS exception handling already provides the error context.
- Prefer logging only write/mutation endpoints in controllers; read endpoints usually do not need controller-level logs.
- Avoid verbose debug logs on simple read endpoints (`list`, `get`, `getData`, `getSecrets`) unless investigating a specific bug.
- Omit stale commented-out decorators in controller examples (`@UseGuards`, `@RequireAdminPermission`) unless preserving intentional documentation is required.
- Don't rely on `LoggerModule` being imported in test files unless explicitly required; unit tests already use `Logger`.
