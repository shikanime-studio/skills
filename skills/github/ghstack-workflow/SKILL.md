---
name: ghstack-workflow
description:
  "Stacked PRs with ghstack + Jujutsu (jj). Create, update, land, and manage
  stacked pull requests where 1 commit == 1 PR. Use when user mentions stack,
  stacked PR, ghstack, or jj squash for PR updates."
version: 1.0.0
author: Operator 21O
license: Apache-2.0
platforms: [linux, macos]
metadata:
  hermes:
    tags: [GitHub, ghstack, Stacked PRs, Jujutsu, jj, PR Workflow]
    related_skills: [github-auth, github-pr-workflow, github-repo-management]
---

# ghstack + jj: Stacked PR Workflow

Manage stacked pull requests using `ghstack` (or `gh stack`) with Jujutsu (`jj`)
as the local VCS. Each commit becomes exactly one PR.

## Core Concept

```text
1 commit  ==  1 PR
N commits ==  N PRs  (called a "stack")
```

Each PR's base is the commit below it. Reviewers see only that layer's diff. The
bottom PR targets `main` (or trunk).

```text
commit-3  -> PR #3 (base: commit-2)  <- top
commit-2  -> PR #2 (base: commit-1)
commit-1  -> PR #1 (base: main)      <- bottom
─────────────
main (trunk)
```

## Prerequisites

- `ghstack` installed (`uv tool install ghstack`) or `gh stack` extension
- `jj` installed and repo initialized (`jj git init --colocate` or `jj init`)
- GitHub auth configured (see `github-auth` skill)
- `~/.ghstackrc` configured with token (for `ghstack` variant)

## Workflow: Create a Stack

```bash
# 1. Start from main
jj bookmark set main -r @-   # ensure main is at desired base

# 2. Make changes and commit each logical unit separately
jj commit -m "Add auth middleware"
jj commit -m "Add rate limiter"
jj commit -m "Add session store"

# 3. Submit the entire stack as PRs
ghstack
# OR: gh stack submit
```

`ghstack` pushes branches and creates one PR per commit. Branch naming:
`gh/<username>/<N>/base`, `gh/<username>/<N>/head`, `gh/<username>/<N>/orig`.

## Workflow: Update a PR in the Stack

```bash
# 1. Edit files as needed
# 2. Squash changes into the TARGET commit (NOT a new commit)
jj squash -r <revision>
# 3. Resubmit the entire stack
ghstack
```

**Critical:** Use `jj squash` (not `jj commit`) to amend an existing commit.
This preserves the 1-commit-1-PR mapping.

### Targeting a specific commit

```bash
# Squash working copy into a specific revision
jj squash -r <rev>

# Or squash a specific revision into another
jj squash <source> -d <destination>
```

## Workflow: Land (Merge) a Stack

```bash
# Land a specific PR (cascading rebase + merge)
ghstack land <PR_URL>
# OR: ghstack land #<PR_NUMBER>
# OR: gh stack merge <PR_NUMBER>
```

**Never** use `gh pr merge` or the GitHub UI merge button on a ghstack PR. This
breaks the stack's base branch tracking and produces `[ghstack-poisoned]`
commits.

## Workflow: Rebase a Stack onto Updated Main

```bash
# Fetch latest main, then rebase the entire stack
jj rebase -d main
# Then resubmit
ghstack
```

**Never** `git merge` into a ghstack branch — ghstack will error because each
commit must remain a separate PR.

## Workflow: Checkout an Existing Stack

```bash
# By PR number
ghstack checkout <PR_NUMBER>
# OR: gh stack checkout <PR_NUMBER>
```

## Workflow: Sync PR Descriptions

```bash
# Sync GitHub PR descriptions back to local commit messages
ghstack sync
```

## Branch Structure (Internal)

For each commit N in a stack, ghstack creates three branches:

| Branch           | Purpose                                                   |
| ---------------- | --------------------------------------------------------- |
| `gh/user/N/base` | Base branch (like `main`). Never force-pushed.            |
| `gh/user/N/head` | The actual change. PR targets `base`. Never force-pushed. |
| `gh/user/N/orig` | Exact local commit. Not visible on GitHub.                |

## Pitfalls

1. **Never `gh pr merge` on a ghstack PR.** Always use `ghstack land`. The UI
   merge button breaks base branch tracking.

2. **Never force-push ghstack branches manually.** ghstack manages branch
   pointers. Manual force-poisoning corrupts the stack.

3. **Never `git merge` into a ghstack branch.** Use `jj rebase` instead. Merge
   commits break the 1-commit-1-PR invariant.

4. **Use `jj squash` not `jj commit` for updates.** A new commit means a new PR.
   Squashing preserves the existing PR.

5. **Rebase before resubmit.** If `main` has moved, rebase your stack first:
   `jj rebase -d main` then `ghstack`.

6. **Stack order matters.** Commits are stacked in topological order (oldest =
   bottom). Reorder with `jj rebase -r <rev> -d <target>` before submitting.

7. **`ghstack` vs `gh stack`:** `ghstack` (ezyang) is the Python CLI with
   `~/.ghstackrc`. `gh stack` (GitHub official) is a Go-based `gh` extension
   with `.git/gh-stack` metadata. They are NOT compatible — pick one per repo.

8. **Splitting a stack into separate PRs.** When asked to split changes into
   individual PRs (1 commit == 1 PR), use `ghstack submit` for each commit. If
   `ghstack submit` fails mid-stack (e.g., timeout, duplicate commit error),
   push remaining commits as separate branches and create PRs manually:

   ```bash
   # Push each commit to its own branch
   git push origin <commit-sha>:refs/heads/fix/<topic>
   # Create PRs manually, basing each on the previous PR's head branch
   gh pr create --title "..." --head fix/<topic> --base <prev-pr-head-branch>
   ```

   This preserves the stack relationship without relying on ghstack's internal
   tracking.

## Quick Reference

| Task              | Command                                 |
| ----------------- | --------------------------------------- |
| Create stack      | `jj commit -m "..."` x N → `ghstack`    |
| Update PR         | edit → `jj squash -r <rev>` → `ghstack` |
| Land PR           | `ghstack land <URL>`                    |
| Rebase stack      | `jj rebase -d main` → `ghstack`         |
| Checkout PR       | `ghstack checkout <N>`                  |
| Sync descriptions | `ghstack sync`                          |
| View stack        | `jj log -r main..@`                     |
