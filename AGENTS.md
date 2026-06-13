# Skills

A curated catalog of self-improved agent skills for Hermes and compatible
agents. Each skill lives in its own directory with a `SKILL.md`.

**Language:** Markdown (SKILL.md)

## Structure

- `skills/` — Individual skill directories, each containing a `SKILL.md`
- `README.md` — Installation and usage documentation

## Commit Style

- Plain-text capitalized title, no conventional-commit prefix
- Body with labels: `Design:`, `Related:`, `Closes #`
- Keep Markdown lines wrapped at 80 columns and run `nix fmt` before shipping

## Stack

- 1 commit == 1 PR via ghstack
- Amend + `ghstack` to resubmit
- `ghstack land` on head PR to land the entire stack
- Never `gh pr merge` (creates poisoned commits)
- Never force-push ghstack branches
- ghstack only works on HEAD commit chains, not detached HEADs

## Protect `main`

- Require 1 approving review
- Require linear history (no merge commits)
- Require signed commits
- Squash+rebase merge only

## Adding a New Skill

1. Create a directory under the appropriate category
2. Write a `SKILL.md` following the
   [skill authoring](https://hermes-agent.nousresearch.com/docs) format
3. Add the skill to the catalog table in `README.md`
4. Commit with a descriptive message

## Skill Categories

- `autonomous-ai` — Agent orchestration and delegation
- `devops` — Infrastructure, Nix, Kubernetes, CI/CD
- `github` — GitHub workflows, issues, PRs
- `productivity` — Communication, documents, automation
- `reconnaissance` — Domain intelligence, OSINT
- `software-dev` — Development workflows, debugging, testing
- `vcs` — Version control systems

_Each skill must include a valid `SKILL.md` with YAML frontmatter. Test against
the target agent before submitting_
