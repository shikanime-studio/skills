# Stacked PR Workflow — Session Notes

## Pattern: Splitting Changes into Stacked PRs

When the user asks to split multiple improvements into separate PRs (1 commit ==
1 PR):

1. **Create each change as a separate commit** on a feature branch
2. **Submit the first commit** with `ghstack submit`
3. **For subsequent commits**, either:
   - Continue using `ghstack submit` (if the stack is clean)
   - Or push commits as separate branches and create PRs manually via
     `gh pr create`

## When ghstack submit fails mid-stack

If `ghstack submit` times out or errors (e.g., "commit occurs twice in your
local stack"):

1. Push each remaining commit to its own branch:

   ```bash
   git push origin <commit-sha>:refs/heads/fix/<topic>
   ```

2. Create PRs manually, basing each on the previous PR's head branch:

   ```bash
   gh pr create --title "..." --head fix/<topic> --base <prev-pr-head-branch>
   ```

3. The first PR in the stack should target `main` (or the base branch)

## Example from shikanime-studio/actions

5 PRs created as a stack:

- PR #224: README fix → base: `main`
- PR #225: Backport fix → base: `gh/shikanime/11/head` (PR #224's head)
- PR #226: Update fix → base: `gh/shikanime/12/head` (PR #225's head)
- PR #227: Land fix → base: `gh/shikanime/13/head` (PR #226's head)
- PR #228: Workflow dispatch → base: `main` (independent)

## Key Rules

- 1 commit == 1 PR (user preference)
- Never combine unrelated changes into a single PR
- Each PR should be independently reviewable
- Stack order: bottom PR targets `main`, each subsequent PR targets the previous
  PR's head branch
- Use `ghstack land` on the head PR to land the entire stack
- Never `gh pr merge` on a ghstack PR
