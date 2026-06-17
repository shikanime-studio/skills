# Composite Action Bug Patterns

## Step Output Reference Mismatch

A step sets an output via `echo "name=value" >> "$GITHUB_OUTPUT"` but has no
`id:`. A later step references it via `steps.<id>.outputs.<name>` — this will
always resolve to empty.

**Example from session (2026-06-13):** `skaffold/integration/action.yaml` — the
first step sets `platform=...` output but has no `id:`. Line 25 references
`steps.skaffold.outputs.platform`, but `id: skaffold` is on the _second_ step
(line 27). The output is never addressable.

**Fix:** Add `id: <name>` to the step that sets the output, and reference it
consistently:

```yaml
- id: platform
  run: |
    echo "platform=$VALUE" >> "$GITHUB_OUTPUT"
- run: |
    echo "${{ steps.platform.outputs.platform }}"
```

## Undefined Shell Array Variables

Using `${args[@]}` in a composite action step without declaring/initializing the
array first. This passes a literal empty string to the command.

**Example from session (2026-06-13):** `skaffold/integration/action.yaml` lines
30-31 — `skaffold build "${args[@]}"` and `skaffold render "${args[@]}"`
reference an `args` variable that was never set.

**Fix:** Either initialize the array from an input, or remove the array
expansion:

```yaml
# Option A: initialize from input
args: ${{ inputs.build-args }}

# Option B: just call the command directly
run: skaffold build
```

## Unsupported Input to Composite Action

Passing an input to a composite action that doesn't declare that input. The
input is silently ignored.

**Example from session (2026-06-13):** `.github/workflows/cleanup.yaml` passes
`pull-request-url: ${{ github.event.pull_request.html_url }}` to
`shikanime-studio/actions/cleanup@v9`, but `cleanup/action.yaml` only declares
`github-token` and `head-ref` inputs.

**Fix:** Check the action's `action.yaml` inputs section before passing custom
inputs. Remove any undeclared inputs.

## Inconsistent Environment Variable Naming

Using `GITHUB_TOKEN` in one action and `GH_TOKEN` in another for the same
purpose. While `gh` CLI accepts both, inconsistency causes confusion and
potential auth failures in environments where only one is set.

**Example from session (2026-06-13):** `update/action.yaml:49` uses
`GITHUB_TOKEN` while all other actions in the repo use `GH_TOKEN`.

**Fix:** Standardize on `GH_TOKEN` across all actions (matches `gh` CLI
convention).

## Verification Protocol

When reviewing composite actions:

1. Check every `steps.<id>.outputs.<name>` reference — verify the step with that
   `id:` exists and sets that output
2. Check every `${array[@]}` expansion — verify the array is initialized
3. Check every `with:` input against the action's declared inputs
4. Check env var naming consistency across actions in the same repo
