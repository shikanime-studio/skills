# Jujutsu Workflow for Conflict Resolution

## Conflict Marker Format

Jujutsu conflict markers differ from git:

```text
<<<<<<< conflict 1 of 2
+++++++ revision "commit message" (destination)
import { Foo } from './bar'
%%%%%%% diff from: mlqsvzzo abc123 (parents of rebased revision)
\\\\\\\\\\ to: mlqsvzzo def456 (rebase destination)
-import { Foo } from './old-path'
-import { Baz } from './qux'
+++++++ revision "other commit" (rebased revision)
import { Foo } from './new-path'
>>>>>>> conflict 1 of 2 ends
```

## Resolution Strategy

1. Understand which side's changes to keep (destination vs rebased revision)
2. Write the clean resolved file directly with `write_file`
3. Verify with `cat` or `read_file` -- `write_file` can silently revert
4. `jj` auto-tracks changes -- no `git add` needed

## Key Commands

| Command                         | Purpose                        |
| ------------------------------- | ------------------------------ |
| `jj status`                     | Check working copy state       |
| `jj log`                        | View commit history            |
| `jj log -r 'heads(trunk()..@)'` | Show commits on current branch |

## Common Pitfalls

- **Stale LSP diagnostics**: After writing resolved files, LSP may still show
  old errors. The file on disk is correct; the LSP just needs time to re-index.
- **write_file silently failing**: Always verify the file content after writing.
  If conflict markers remain, retry.
- **Test failures after resolution**: Conflict resolution often leaves type
  mismatches or renamed fields. Always run tests after resolving.
