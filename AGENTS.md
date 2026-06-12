# Agents

This repository is a catalog of self-improved agent skills for Hermes and
compatible agents. Each skill lives in its own directory with a `SKILL.md`.

## Coding Style

- Keep `SKILL.md` files focused and actionable.
- Use YAML frontmatter with `name` and `description` fields.
- Include a `## Trigger` section when the skill has specific activation
  conditions.
- Include a `## Pitfalls` section for known issues and workarounds.
- Keep lines wrapped at 80 columns.
- Run `nix fmt` before shipping.

## Adding a New Skill

1. Create a directory under the appropriate category.
2. Write a `SKILL.md` following the
   [skill authoring](https://hermes-agent.nousresearch.com/docs) format.
3. Add the skill to the catalog table in `README.md`.
4. Commit with a descriptive message.

## Skill Categories

- `autonomous-ai` — Agent orchestration and delegation
- `devops` — Infrastructure, Nix, Kubernetes, CI/CD
- `github` — GitHub workflows, issues, PRs
- `productivity` — Communication, documents, automation
- `reconnaissance` — Domain intelligence, OSINT
- `software-dev` — Development workflows, debugging, testing
- `vcs` — Version control systems
