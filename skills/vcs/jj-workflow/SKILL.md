---
name: jj-workflow
description: "Jujutsu (jj) daily workflow — init, commit, push, rebase, resolve conflicts, and interop with git hosting (GitHub/GitLab). Use when the user asks about jj operations, or when any repo workflow is needed (jj is the preferred VCS)."
version: 1.0.0
author: Hermes Agent
license: Apache-2.0
platforms: [linux, macos]
metadata:
  hermes:
    tags: [jj, jujutsu, vcs, git, workflow, commit, push, rebase, conflict]
    related_skills: [github-pr-workflow]
---

# Jujutsu (jj) Daily Workflow

jj is the preferred VCS. Git is acceptable when needed for interop.

## Commit Style

Commit messages are **full short sentences**, not conventional commits:

```
jj commit -m "Fix some stuff about that"
jj commit -m "Update the firewall rules for the new subnet"
jj commit -m "Add missing netpol for vaultwarden"
```

Not `fix: add foo` — just plain sentences. One commit per logical fix. No body, no trailers.

## Init

```bash
# New repo (colocated with git for GitHub/GitLab interop)
jj git init --colocate

# Clone an existing repo
jj git clone git@github.com:org/repo.git ~/Source/Repos/github.com/org/repo
# or
gh repo clone org/repo ~/Source/Repos/github.com/org/repo
cd ~/Source/Repos/github.com/org/repo
```

Clone pattern follows XDG: `~/Source/Repos/<domain>/<org>/<repo>`

## Daily Workflow

```bash
# Check status
jj status

# See recent log
jj log -n 20

# See what changed in working copy
jj diff

# Commit everything in working copy
jj commit -m "Describe what changed"

# Commit only specific files
jj commit -m "Fix the thing" -- path/to/file1 path/to/file2

# Amend the last commit (auto-squash into @)
jj commit --amend -m "Better description"
```

## Branching & Bookmarks

```bash
# Create a bookmark (branch) for a change
jj bookmark create fix-thing -r @-
# or shorthand
jj bookmark c fix-thing -r @-

# List bookmarks
jj bookmark list

# Push a bookmark to remote
jj git push --bookmark fix-thing
# Push all bookmarks
jj git push --all

# Track a remote bookmark
jj bookmark track fix-thing@origin

# Delete a bookmark
jj bookmark delete fix-thing
```

## Stacking Changes

jj doesn't need explicit stacks — just build on top of the working copy parent:

```bash
# Make first change
jj commit -m "Add feature A"

# Make second change (automatically on top of feature A)
jj commit -m "Add feature B"

# Go back to an earlier commit and make a sibling change
jj edit <rev>  # or jj new <rev>
# ... make changes ...
jj commit -m "Fix edge case in feature A"
```

Use `jj split` to split one change into two:

```bash
jj split <paths-or-interactive>
```

## Rebasing

```bash
# Rebase current change onto main
jj rebase -d main

# Rebase a range
jj rebase -s <start> -d <dest>

# Rebase all descendants of a commit
jj rebase -r <rev> -d <dest>
```

## Conflict Resolution

```bash
# jj marks conflicts; edit the files and resolve them
jj resolve  # interactive conflict resolver

# Or manually fix files, then:
jj squash  # fold resolution into the conflicted commit

# See conflicted files
jj status
```

## Undo / Backtrack

```bash
# Undo the last operation (safe — jj keeps everything)
jj undo

# Abandon a commit (removes it from the graph)
jj abandon <rev>

# Restore a previous state
jj restore <rev> -- <files>
```

## Git Interop

```bash
# Fetch from remote
jj git fetch

# Push to remote
jj git push

# Create a bookmark from a GitHub PR
jj bookmark create pr-123 -r <rev>
jj git fetch
jj bookmark track pr-123@origin

# Pull (note: jj doesn't have a native pull — fetch + rebase)
jj git fetch
jj rebase -d main@origin
```

## New Work (Start Fresh)

```bash
# Start a new empty change on top of @
jj new

# Start from a specific revision
jj new main
jj new <commit-id>

# Describe the working commit
jj describe -m "What I'm about to do"
```

## Squashing

```bash
# Squash changes from @- into @
jj squash  # interactive
jj squash --from @- --to @  # explicit

# Squash everything since main into one commit
jj squash --from main
```

## Log & Inspection

```bash
# Compact log with bookmarks
jj log -T 'commit_id.short() ++ " " ++ description.first_line() ++ " " ++ bookmarks'

# Full diff of a commit
jj show <rev>

# Show changes between two revs
jj diff --from main --to @

# See which changes are not on main
jj log -r 'main..@'
```

## Access Control (Repository Rules)

For repos with `.jj/repos` ACL config:

```bash
# Check repo-level permissions
jj config list repo
```

## Common Pitfalls

1. **`jj git push` fails with "No matching bookmarks"**: Create the bookmark first with `jj bookmark create <name> -r @-`, then push with `jj git push --bookmark <name>`.

2. **Conflicts on rebase**: Expected when stacking changes. Resolve with `jj resolve` or manually edit then `jj squash`.

3. **Detached HEAD**: If `jj show @` has no commit_id, use `jj new` to create a new commit before working.

4. **Accidental abandon**: Use `jj undo` immediately. jj keeps everything until the `git gc` equivalent runs.

5. **Commit scope**: `jj commit` commits everything in the working copy. To commit selectively, use `jj commit -- <paths>` or split first.

## See Also

- `references/ghstack-integration.md` — how jj interacts with ghstack for stacked PRs
