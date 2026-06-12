# Skills

A curated catalog of self-improved agent skills extracted from the local Hermes
instance. These are custom-created or substantially modified skills that may be
useful to share among agents.

## What's Here

Skills are organized by category. Each skill is a self-contained directory with
a `SKILL.md` that follows the
[Agent Skills](https://hermes-agent.nousresearch.com/docs) format.

### Custom Skills (Original Creations)

These skills were created from scratch and are not part of any upstream hub:

| Skill | Category | Description |
|-------|----------|-------------|
| `apple-notes` | apple | Manage Apple Notes via memo CLI |
| `apple-reminders` | apple | Apple Reminders via remindctl |
| `claude-code` | autonomous-ai | Delegate coding to Claude Code CLI |
| `codex` | autonomous-ai | Delegate coding to OpenAI Codex CLI |
| `kanban-worker` | autonomous-ai | Pitfalls and examples for Hermes Kanban workers |
| `github-auth` | github | GitHub auth setup: HTTPS tokens, SSH keys, gh CLI |
| `github-code-review` | github | Review PRs: diffs, inline comments via gh or REST |
| `github-ghstack-workflow` | github | Stacked PRs with ghstack + jj: create, update, land stacks |
| `github-issues` | github | Create, triage, label, assign GitHub issues |
| `github-repo-management` | github | Clone/create/fork repos; manage remotes, releases |
| `home-manager-pitfalls` | devops | Home Manager module gotchas and silent failures |
| `nix-module-bulk-edit` | devops | Bulk-edit Nix module files using reliable patterns |

### Improved Skills (Substantially Modified)

These skills have been improved from their upstream versions:

| Skill | Category | Description |
|-------|----------|-------------|
| `dogfood` | dogfood | Exploratory QA of web apps |
| `security-best-practices` | security | Language/framework security best practices |
| `security-threat-model` | security | Repository-grounded threat modeling |
| `yuanbao` | yuanbao | Yuanbao groups: @mention, query info/members |
| `hermes-agent-skill-authoring` | hermes | Author in-repo SKILL.md |
| `plan` | hermes | Write implementation plans |
| `writing-plans` | hermes | Write implementation plans (obra/superpowers) |
| `spike` | hermes | Throwaway experiments |
| `systematic-debugging` | hermes | 4-phase root cause debugging |
| `test-driven-development` | hermes | Enforce RED-GREEN-REFACTOR |
| `subagent-driven-development` | hermes | Execute plans via delegate_task subagents |
| `multi-repo-review` | hermes | Review multiple repositories in parallel |
| `requesting-code-review` | hermes | Pre-commit review workflow |
| `dependency-upgrades` | hermes | Upgrade pnpm monorepo dependencies |
| `vitest-configuration` | hermes | Troubleshoot Vitest config issues |
| `python-debugpy` | hermes | Debug Python via pdb + debugpy |
| `node-inspect-debugger` | hermes | Debug Node.js via Chrome DevTools Protocol |
| `debugging-hermes-tui-commands` | hermes | Debug Hermes TUI slash commands |
| `sonarqube-refactoring` | hermes | Fix SonarQube quality gate issues |
| `nix-docker-builds` | hermes | Build Docker images with Nix |
| `nestjs-build-fixing` | nestjs | Fix NestJS + TypeScript build errors |
| `nestjs-logging` | nestjs | Add logging to NestJS modules |
| `nestjs-module-migration` | nestjs | Migrate legacy backend modules to NestJS |
| `cpn-console-plugins` | cpn | Develop cloud-pi-native console plugins |
| `cpn-documentation` | cpn | Write cloud-pi-native documentation |

## Installation

### Manual Installation

Copy individual skill directories to your `~/.hermes/skills/` directory:

```bash
cp -r skills/github/github-auth ~/.hermes/skills/github/
```

### With Nix

This repo uses [devlib](https://github.com/shikanime-studio/devlib) for
development tooling.

```bash
nix develop
# or
direnv allow
```

## Development

```bash
nix develop
```

## License

MIT — See [LICENSE](./LICENSE) for details.
