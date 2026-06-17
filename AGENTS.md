# Skills

A curated catalog of self-improved agent skills for
[Hermes](https://hermes-agent.nousresearch.com/docs) and compatible agents. Each
skill lives in its own directory with a `SKILL.md`.

**Language:** Markdown (SKILL.md)

## Structure

- `skills/` — Individual skill directories, each containing a `SKILL.md`
  - `apple/` — Apple ecosystem CLI tools
  - `autonomous-ai/` — Agent orchestration and delegation
  - `cpn/` — Cloud Pi Native console plugins
  - `devops/` — Infrastructure, Nix, Kubernetes, CI/CD
  - `dogfood/` — Exploratory QA
  - `github/` — GitHub workflows, issues, PRs
  - `hermes/` — Hermes Agent operations
  - `nestjs/` — NestJS development
  - `security/` — Security review and threat modeling
  - `vcs/` — Version control systems
  - `yuanbao/` — Yuanbao group chat
- `README.md` — Installation and usage documentation
- `package.json` — npm package manifest with `agents` field for skill discovery

## Installation

```bash
# Via npx skills (Claude Code, Codex, Cursor, OpenCode, …)
npx skills add shikanime-studio/skills -g -y

# Via Hermes tap
hermes skills tap add shikanime-studio/skills

# Via npm
npm install @shikanime-studio/skills
```

## Adding a New Skill

1. Create a directory under the appropriate category in `skills/`
2. Write a `SKILL.md` following the
   [skill authoring](https://hermes-agent.nousresearch.com/docs) format with
   valid YAML frontmatter
3. Add the skill to the catalog table in `README.md`
4. Test against the target agent before submitting
5. Commit with a descriptive message

## Commit Style

- Plain-text capitalized title, no conventional-commit prefix
- Body with labels: `Design:`, `Related:`, `Closes #`
- Keep Markdown lines wrapped at 80 columns and run `nix fmt` before shipping

## Stack

- 1 commit == 1 PR via ghstack (1 commit is 1 logical atomic change)
- Split work into stacked PRs to keep each PR small and reviewable
- To pull down an existing stack: `ghstack checkout <PR_NUMBER>`
- To update a PR: edit files, then `jj squash` (or `git commit --amend`) into
  the **target commit** of the stack — the one that PR represents
- Resubmit with `ghstack` after squashing
- `ghstack land` on the head PR to land the entire stack
- Never `gh pr merge` (creates poisoned commits)
- Never force-push ghstack branches

## Protect `main`

- Require 1 approving review
- Require linear history (no merge commits)
- Require signed commits
- Squash+rebase merge only

_Licensed under Apache-2.0. Each skill must include a valid `SKILL.md` with YAML
frontmatter. Always use worktrees when making changes._
