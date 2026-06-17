# PR 2182 — archiveProject Deletion Order Investigation

## Problem

`archiveProject` in `apps/server/src/resources/project/business.ts` calls
`deleteAllRepositoryForProject` and `deleteAllEnvironmentForProject` BEFORE
`hook.project.delete(projectId)`.

```ts
const [projectDb, ..._] = await Promise.all([
  prisma.project.findUniqueOrThrow(...),
  deleteAllRepositoryForProject(projectId),
  deleteAllEnvironmentForProject(projectId),
])
const { results, project } = await hook.project.delete(projectId)
```

This means all plugins receive an empty `project.repositories` and
`project.environments` array during the `deleteProject` hook.

## Affected Plugins

### Argocd (`plugins/argocd/src/functions.ts`)

`deleteProject` (step `main`):

- Calls `getDistinctZones(project)` which reads `project.clusters[*].zone.slug`
- Lists files from GitLab infra project via `gitlabApi.listFiles`
- Deletes all matched values.yaml files via `gitlabApi.commitDelete`

Currently works because `deleteProject` discovers files from GitLab rather than
using `project.environments` for the delete list. However, if a future refactor
makes it fall back to deriving paths from DB env rows, it will break silently.

### Other Plugins

keycloak (`post`): uses `project.slug` only — no impact vault (`post`): uses
`project.slug` only — no impact harbor (`main`): uses `project.slug` only — no
impact nexus (`main`): uses `project.slug` only — no impact

`getHookProjectInfos` includes `repositories`, `environments`, `clusters`,
`owner`, `roles`, `members`, `plugins` — so hooks get a populated payload
regardless.

## Recommended Fix

Move DB deletion AFTER `hook.project.delete()`:

```ts
export async function archiveProject(projectId, requestor, requestId) {
  const projectDb = await prisma.project.findUniqueOrThrow({...})

  if (projectDb.locked) ...
  if (projectDb.status === 'archived') ...

  const { results, project } = await hook.project.delete(projectId)
  // log results …

  if (results.failed) return new Unprocessable422(...)

  await Promise.all([
    deleteAllRepositoryForProject(projectId),
    deleteAllEnvironmentForProject(projectId),
  ])

  return null
}
```

This preserves full project state for all delete hooks with zero risk to the
archive semantics (the project row itself is untouched).

## Failure Semantics After Reorder

If `hook.project.delete()` fails:

- DB records for repos/envs are still intact → retry is safe
- Some plugins may have already deleted infra externally (e.g. ArgoCD values)
  while others didn't run → partial infra state possible but resumable

If hook succeeds:

- All infra cleaned
- DB purged atomically after — clean end state

## Relevant File Map

- `apps/server/src/resources/project/business.ts:216` — `archiveProject`
  definition
- `apps/server/src/resources/project/queries.ts:211` — `getHookProjectInfos`
  (includes repos, envs, clusters)
- `apps/server/src/utils/hook-wrapper.ts:109` — `project.delete` wrapper
- `plugins/argocd/src/functions.ts:238` — `deleteProject` step
- `plugins/keycloak/src/index.ts:23` — deleteProject `post`
- `plugins/vault/src/index.ts:25` — deleteProject `post`
- `plugins/harbor/src/index.ts:10` — deleteProject `main`
- `plugins/nexus/src/index.ts:10` — deleteProject `main`
