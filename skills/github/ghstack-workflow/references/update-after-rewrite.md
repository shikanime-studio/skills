# ghstack: Updating PRs After History Rewrite

## Problem

You need to update an existing ghstack PR with formatting fixes or other
changes, but the commits were squashed/rebased outside of ghstack's workflow.
Running `ghstack submit` creates NEW PRs instead of updating the existing one.

## Root Cause

ghstack tracks PRs via commit message metadata trailers (`[ghstack ...]`). When
you squash, rebase, or rewrite commits, the new commits lack this metadata.
ghstack treats them as unrelated to the original PR.

## Solution

```bash
# 1. Restore the original PR branch with ghstack metadata
ghstack checkout <PR_NUMBER>

# 2. Make your changes (edit files, run formatters, etc.)

# 3. Squash changes into the TARGET commit (preserves ghstack metadata)
jj squash -r <target-revision>

# 4. Submit to update the existing PR
ghstack submit
```

## What NOT To Do

- `git push --force` to `gh/user/N/head` — updates the branch but NOT the PR (no
  metadata -> ghstack can't associate it with the existing PR)
- `ghstack submit` on commits without metadata — creates duplicate PRs
- `gh pr merge` or GitHub UI merge — poisons the stack

## If You Already Created Duplicate PRs

```bash
# Close the duplicate PRs
gh pr close <DUPLICATE_PR_NUMBER>

# Restore original branch state
git push origin <ORIGINAL_SHA>:gh/user/N/head --force
git push origin <ORIGINAL_BASE_SHA>:gh/user/N/base --force

# Then redo properly with ghstack checkout -> squash -> ghstack submit
```

## Formatting Fixes Workflow

When updating a PR with `nix fmt` or other formatters that produce many changes:

```bash
# 1. Checkout the PR to get ghstack metadata
ghstack checkout <PR_NUMBER>

# 2. Run formatter and check results
nix fmt

# 3. If formatter reports failures, fix them:
#    - Add markdownlint-disable directives to security reference files with
#      many long lines (MD013, MD034, MD075)
#    - For MD076/MD031 conflicts, disable MD076
#    - See references/formatting-conflicts.md for common patterns

# 4. Commit formatting fixes
git add -A
git commit -m "Fix markdownlint issues across skill files

Signed-off-by: <Your Name> <<email>>"

# 5. Squash into the PR's target commit
jj squash -r @-

# 6. Submit to update existing PR (preserves metadata)
ghstack submit
```

### Common Formatting Conflicts in Skill Files

| Rule  | Cause                           | Fix                                        |
| ----- | ------------------------------- | ------------------------------------------ |
| MD013 | Long lines in code blocks       | Add `<!-- markdownlint-disable MD013 -->`  |
| MD034 | Bare URLs in reference sections | Auto-fixed by nix fmt, or disable MD013    |
| MD075 | Pipe characters in link titles  | Disable MD075 for security reference files |
| MD076 | Blank lines before code blocks  | Disable MD076 when MD031 also triggers     |
| MD055 | Table pipe style conflicts      | Disable both MD055/MD056 to break cycles   |
| MD056 | Table row cell count mismatch   | Usually resolves when MD055 disabled       |
