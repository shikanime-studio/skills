# Console → GitLab Role Mappings

## Project Roles (scope projet)

| Rôle Console                        | Droits GitLab (groupe du projet) |
| ----------------------------------- | -------------------------------- |
| Administrateur (`/console/admin`)   | Maintainer                       |
| DevOps (`/console/devops`)          | Developer                        |
| Développeur (`/console/developer`)  | Developer                        |
| Lecture seule (`/console/readonly`) | Reporter                         |
| Propriétaire du projet (Console)    | Owner                            |

## Platform Roles (scope Console)

| Rôle Console        | Droits GitLab         |
| ------------------- | --------------------- |
| `/console/admin`    | Administrateur GitLab |
| `/console/readonly` | Auditeur GitLab       |

## Version-Scoped Notes

### 9.16.x

- **Retrocompat phase**: GitLab still assigns **Developer** as the default role
  when adding a member. The _Lecture seule_ role (`/console/readonly` →
  Reporter) has no practical effect because Developer takes precedence.
- Future evolution: GitLab default will switch to **Guest**, allowing _Lecture
  seule_ to apply correctly.

### Authoritative Provisioning (current)

- Roles are provisionned authoritatively in integrated services (GitLab).
- Console takes full control of membership: untracked members are **deleted**,
  Console-assigned members are **added**.
- Manual membership changes in GitLab are overwritten on sync.
- Purpose: prevent drift between Console and external services.
