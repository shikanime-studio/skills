---
name: cpn-console-plugins
description:
  Develop, debug, and fix plugins for the cloud-pi-native console. Covers the
  plugin/hook lifecycle, GitLab deferred commit pattern, shared API instances,
  and ArgoCD plugin specifics. Triggered when working on any console plugin
  (argocd, gitlab, vault, keycloak, etc.), debugging hook execution, or
  modifying plugin functions.
---

# CPN Console Plugin Development

## Hook Execution Model

Hooks execute in sequential steps: `pre` → `main` → `post`. Within each step,
all registered plugins run **in parallel** via `Promise.allSettled`. If any
plugin fails during a step, the `revert` step runs for rollback.

Key files:

- `packages/hooks/src/hooks/hook.ts` — core execution engine
- Each plugin registers hooks in its `index.ts` via `subscribedHooks`

## Shared API Instances

All plugins in a hook execution share the same API instances. The GitLab plugin
creates `GitlabProjectApi` or `GitlabZoneApi` via the `api` factory in its
`index.ts`. Other plugins (ArgoCD, Vault, etc.) access these via
`payload.apis.gitlab`.

This means state (like pending commits) is shared across plugins within a single
hook execution.

## GitLab Deferred Commit Pattern

**Critical:** `commitDelete` and `commitCreateOrUpdate` do NOT immediately call
the GitLab API. They queue actions into `this.pendingCommits` on the
`GitlabProjectApi` instance.

The actual GitLab API call happens later when `commitFiles()` is called
(registered as the `post` step). This batches all file operations into a single
commit.

Flow:

1. Plugin `main` step: `commitDelete`/`commitCreateOrUpdate` → queues into
   `pendingCommits`
2. Plugin `post` step: `commitFiles()` → calls `Commits.create` with all queued
   actions

**Implication:** If you call `commitDelete` and the `post` step fails or is not
registered, the deletions never happen. Always verify the `commitFiles`
post-step is registered for the hook.

## ArgoCD Plugin

The ArgoCD plugin manages `values.yaml` files in GitLab infra projects. It does
NOT interact with the ArgoCD API directly — it's purely GitOps through file
management.

Key functions in `plugins/argocd/src/functions.ts`:

- `upsertProject` — creates/updates values.yaml for each env, then calls
  `removeInfraEnvValues` for stale-file cleanup
- `deleteProject` — deletes all `values.yaml` infra files for the project. Now
  delegates to `removeInfraEnvValues` for surgical deletion consistent with
  `upsertProject`. Does NOT use a separate recursive-wipe helper.
- `removeInfraEnvValues` — shared helper. For each zone, lists files under
  `project.name/`, builds a `neededFiles` set from
  `getValueFilePath(project, cluster, env)` for `project.environments`, and
  deletes only `values.yaml` blobs whose path is _not_ in that set. Works for
  both upsert (evict stale envs) and delete (all envs gone → deletes
  everything).

Zone lookup ~~uses `getDistinctZones(project)`~~ — `removeInfraEnvValues`
already iterates zones via `getDistinctZones`, so `deleteProject` no longer
needs its own zone list.

## Project Deletion Order (archiveProject)

**Key invariant: hooks first, DB purge second.**

`archiveProject` in `apps/server/src/resources/project/business.ts` MUST call
`hook.project.delete(projectId)` BEFORE `deleteAllRepositoryForProject` and
`deleteAllEnvironmentForProject`. Otherwise plugins receive empty
`project.repositories` and `project.environments` arrays.

```ts
// Correct: hooks first, then empty DB
const projectDb = await prisma.project.findUniqueOrThrow({
  where: { id: projectId },
  include: { members: { include: { user: true } } },
});

const { results, project } = await hook.project.delete(projectId);
// log + handle failure …

await Promise.all([
  deleteAllRepositoryForProject(projectId),
  deleteAllEnvironmentForProject(projectId),
]);
```

**Impact by plugin when order is wrong:**

| Plugin     | step   | DB data needed                                                                                      | Order-sensitive?                                               |
| ---------- | ------ | --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `argocd`   | `main` | `clusters[*].zone.slug` for `removeInfraEnvValues`; `project.repositories` for infra-repo discovery | Yes — empty env list short-circuits the surgical delete helper |
| `keycloak` | `post` | `project.slug`                                                                                      | No — row itself survives                                       |
| `vault`    | `post` | `project.slug`                                                                                      | No                                                             |
| `harbor`   | `main` | `project.slug`                                                                                      | No                                                             |
| `nexus`    | `main` | `project.slug`                                                                                      | No                                                             |

**Fix history (PR 2182):**

- Moved `deleteAllRepositoryForProject` / `deleteAllEnvironmentForProject` to
  _after_ the hook.
- Refactored `deleteProject` to reuse `removeInfraEnvValues` so delete and
  upsert share one surgical cleanup path and don't diverge.
- Removed the now-unused `getDistinctZones` helper from `deleteProject`.

See `references/2182-deletion-order.md` for the original investigation and
plugin-specific impact matrix.

## Logger Pattern

Console plugins use a shared `@cpn-console/logger` package. Each plugin should
expose a scoped child logger.

```ts
// src/logger.ts
import type { Logger } from "@cpn-console/logger";
import { logger as baseLogger } from "@cpn-console/logger";
export const logger: Logger = baseLogger.child({ plugin: "<plugin-name>" });
```

Dependency:

```json
"@cpn-console/logger": "workspace:^"
```

Conventions:

- Log hook start/done at `info` level in exported step calls.
- Log intermediate state changes at `debug` level.
- Log failures at `error` level with structured fields: `action`,
  `projectSlug|projectId`, and `err`.
- Use `projectSlug` whenever available; avoid bare messages.
- Prefer top-level logs only in exported step calls. Low-value scheduler-style
  `Start`/`Done` around helpers should usually be omitted if they carry no
  useful state.

## Common Pitfalls

- **Environments deleted before hook runs:** In `archiveProject`,
  `deleteAllEnvironmentForProject` runs BEFORE the `deleteProject` hook. So
  `project.environments` is empty during hook execution. Code that depends on
  environments must handle this — or better, move DB deletion after the hook.
- **Trailing slash in listFiles path:** `listFiles` requires the trailing slash
  for directory traversal. Without it, the GitLab API may return empty results,
  causing cleanup to silently delete nothing.
- **commitDelete returns true immediately:** Don't assume the deletion has
  happened when `commitDelete` returns. It's deferred to the `post` step.
- **Partial failure after cleanup:** If a plugin deletes infra during `main` and
  another plugin fails in `post`, the `post` step still runs for all plugins
  (Promise.allSettled style), so deferred commits usually flush. Only a
  catastrophic failure in the `post` step itself prevents that final batch.

## Workspace Verification

After changing a plugin's dependencies, install with:

```bash
pnpm install --filter @cpn-console/<plugin-name>-plugin
```

Verify with plugin-targeted build:

```bash
pnpm --filter @cpn-console/<plugin-name>-plugin run build
```

Monorepo-wide builds can be blocked by unrelated workspaces; prefer targeted
verification for plugin changes.
