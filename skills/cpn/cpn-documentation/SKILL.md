---
name: cpn-documentation
description:
  Write and maintain cloud-pi-native/documentation (VitePress site).
  Admonitions, role mapping warnings, breaking change notes, and French-language
  conventions.
tags: [cloud-pi-native, documentation, vitepress, french]
triggers:
  - editing markdown files under cloud-pi-native/documentation
  - adding warnings, breaking changes, or migration notes to docs
  - cloud-pi-native doc site tasks
---

# cloud-pi-native Documentation

## Conventions

- Documentation is written in **French**.
- VitePress admonition syntax: `:::tip`, `:::warning`, `:::danger` blocks.
- Role mapping tables use the Console → GitLab correspondence format (see
  `references/role-mappings.md`).

## Admonition Usage

| Severity | Syntax       | When to use                                                      |
| -------- | ------------ | ---------------------------------------------------------------- |
| Info     | `:::tip`     | Helpful but non-critical guidance                                |
| Warning  | `:::warning` | Temporary limitations, retrocompat phases, upcoming deprecations |
| Danger   | `:::danger`  | Breaking changes, data loss risk, irreversible behavior          |

### Framing Rules

- **Retrocompat / legacy phases**: frame as intentional temporary behavior for
  compatibility, not as a bug to fix. Use wording like "Phase de
  rétrocompatibilité" or "conservé temporairement pour compatibilité".
- **Breaking changes**: state the concrete impact (what gets deleted,
  overwritten, or stops working) and the mechanism (trigger, sync cycle, etc.).
- Always note the version scope (e.g., `9.16.x`) and the intended future
  evolution.

## Key Locations

| File                           | Content                                                        |
| ------------------------------ | -------------------------------------------------------------- |
| `docs/guide/roles.md`          | Project roles — user-facing role management and GitLab mapping |
| `docs/administration/roles.md` | Platform roles — admin reference, plugin config keys           |

## Pitfalls

- Do not frame retrocompat behavior as a "bug" or "defect". The user corrected
  this: use "phase de rétrocompatibilité" — it is intentional and temporary.
- When adding admonitions near examples or screenshots, place them **before**
  the example so readers see the warning before encountering the illustration.

## See also

- `references/role-mappings.md` — Console → GitLab role correspondence tables
  and version-scoped notes
