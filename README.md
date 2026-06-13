# Skills

A curated catalog of self-improved agent skills for
[Hermes](https://hermes-agent.nousresearch.com/docs) and compatible agents.

These skills are either **original creations** written from scratch, or
**substantially modified** versions of upstream skills adapted for this
agent's workflow. They are not mass-generated — each has been tested in
production use.

## Quick Start

### Install via npx skills (Claude Code, Codex, Cursor, OpenCode, …)

[skills.sh](https://skills.sh/) is the open agent skills registry. Install any
skill from this repo:

```bash
# List available skills
npx skills add shikanime-studio/skills --list

# Install all skills globally
npx skills add shikanime-studio/skills -g -y

# Install a specific skill
npx skills add shikanime-studio/skills --skill github-auth -g

# Install all skills for specific agents
npx skills add shikanime-studio/skills -g -a claude-code -a cursor -y
```

### Install as a Hermes skill source

Add the repo as a tap and configure the external directory:

```bash
# Add as a tap
hermes skills tap add shikanime-studio/skills

# Configure external directory for auto-loading
hermes config set skills.externalDirs \
  '["~/Source/Repos/github.com/shikanime-studio/skills/skills"]'

# Verify loaded skills
hermes skills list
```

### Install individual skills

```bash
# Install a single skill from the tap
hermes skills install shikanime-studio/skills/kanban-worker

# Or copy manually
cp -r skills/github/github-auth ~/.hermes/skills/github/
```

### Install via npm

```bash
# Install as an npm package (skills are bundled via the agents field)
npm install @shikanime-studio/skills

# Then export to your agent's skill directory
npx agents export --target claude
```

The `agents` field in `package.json` and the `skills.json` manifest at the repo
root enable discovery by npm-based skill managers.

## What's Here

Skills are organized by category. Each skill is a self-contained directory with
a `SKILL.md` that follows the [Agent Skills](https://agentskills.io/specification)
specification and is compatible with the
[Hermes format](https://hermes-agent.nousresearch.com/docs).

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
| `github-issues` | github | Create, triage, label, assign GitHub issues |
| `github-repo-management` | github | Clone/create/fork repos; manage remotes, releases |
| `ghstack-workflow` | github | Stacked PRs with ghstack + jj: create, update, land stacks |
| `darwin-host-provisioning` | devops | Provision and restore macOS Nix-Darwin hosts |
| `home-manager-pitfalls` | devops | Home Manager module gotchas and silent failures |
| `nix-module-bulk-edit` | devops | Bulk-edit Nix module files using reliable patterns |
| `system-migration-audit` | devops | Audit and plan system package manager migrations |
| `jj-workflow` | vcs | Jujutsu daily workflow: commit, push, rebase, conflicts |
| `windows-hermes-setup` | hermes | Windows-specific Hermes setup, pitfalls, and workarounds |

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

## Development

```bash
nix develop
```

Format Nix files before committing:

```bash
nix fmt
```

## License

Apache 2.0 — See [LICENSE](./LICENSE) for details.
