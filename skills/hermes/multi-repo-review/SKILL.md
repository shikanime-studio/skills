---
name: multi-repo-review
description: "Review multiple repositories in parallel for code quality, security, configuration issues, and improvements. Use when the user asks to review 2+ repos, audit a set of projects, or compare patterns across repositories."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [code-review, multi-repo, audit, quality, security, parallel]
    related_skills: [requesting-code-review, codebase-inspection, dogfood]
---

# Multi-Repository Code Review

Review multiple repositories in parallel using subagents, then synthesize findings into a prioritized report.

## When to Use

- User asks to review 2+ repositories
- User says "review these repos", "audit this project", "check for issues across"
- User provides a list of repo names or org/repo patterns
- Comparing patterns or consistency across related repos

**Skip for:** single-repo review (use `requesting-code-review` or `codebase-inspection` instead).

## Workflow

### Phase 1: Plan

1. Identify the repos to review. If the user gives names like "shikanime/manifests", use `gh repo clone <org>/<name>`. Clone to the user's preferred source directory (XDG layout, e.g. `~/Source/Repos/github.com/<org>/<repo>`). Do not clone to `/tmp/` for repos the user may want to keep working on.
2. Determine the review scope from user input. Default: code quality, security, configuration, duplication, outdated patterns, missing error handling.
3. Check if repos are already cloned (e.g., from a previous subagent run) before re-cloning.

### Phase 2: Parallel Subagent Dispatch

Use `delegate_task` with a `tasks[]` array — one task per repo. All run concurrently.

```python
delegate_task(
    tasks=[
        {
            "goal": "Clone and review <org>/<repo>. Check for: code quality issues, duplicated code, outdated patterns, potential bugs, configuration inconsistencies, unused imports, missing error handling, and general improvements. Read ALL source files thoroughly. Report findings with specific file paths and line numbers.",
            "context": "Repository: <org>/<repo> on GitHub. Use `gh repo clone <org>/<repo> /tmp/review-<shortname>` to clone it.",
            "toolsets": ["terminal", "file"]
        },
        # ... one per repo
    ]
)
```

**Important:** Limit to 3 concurrent tasks (platform max). If reviewing more than 3 repos, batch them.

### Phase 2.5: Match Commit Style Before Submitting PRs

Before creating PRs from review fixes, check the existing commit style of each repo:

```bash
git log --oneline -20
```

Match the repo's convention exactly. Common patterns in shikanime repos:
- **Short imperative, no body**: `Fix missing netpols for vaultwarden`
- **No sign-off trailers** unless the repo has a commit-msg hook
- **No `Co-Authored-By`** trailers unless explicitly used by the repo
- **One commit per logical fix** — don't bundle unrelated changes
- **Use `ghstack submit`** for PR creation when the repo supports it (the user prefers ghstack over `gh pr create`)

If you already created PRs with the wrong style, close them and redo. The user will notice and ask you to fix it.

### Phase 2.6: Create PRs with ghstack

For each fix, create a separate branch and use ghstack:

```bash
git checkout -b fix/specific-fix
# ... apply changes ...
git commit -m "Short imperative message matching repo style"
git push origin fix/specific-fix
ghstack submit --stack
```

This creates one PR per commit, linked in a stack. Edit PR descriptions afterward with `gh pr edit` to add context about what was fixed and why.

### Phase 3: Verify Subagent Results

Subagents can hit token/iteration limits on large repos. After they complete:

1. **Check each summary for completeness.** If a subagent hit max_iterations or returned an error, clone that repo yourself and do a direct review.
2. **Verify critical claims.** Subagents can produce false positives. For any 🔴 Critical finding, read the source file directly to confirm before reporting to the user.
3. **Fill gaps.** If a subagent's summary is shallow or missing categories, supplement with your own `read_file` passes.

### Phase 4: Synthesize Report

Combine findings across all repos into a single structured report:

```
## Repo Name

### 🔴 Critical
| Issue | File | Detail |
|-------|------|--------|

### 🟡 Medium
...

### 🟢 Low / Refactor
...

### ✅ What's Good
...
```

Then add a **Priority Order for Fixes** section ranking all findings across all repos by severity and impact.

## Review Checklist

For each repo, check:

- [ ] **Code quality**: unused imports, dead code, inconsistent patterns
- [ ] **Duplication**: copy-pasted code that should be shared
- [ ] **Outdated patterns**: deprecated API versions, old package versions
- [ ] **Bugs**: wrong references, undefined variables, missing error handling
- [ ] **Security**: hardcoded secrets, missing auth, overly permissive settings
- [ ] **Configuration**: inconsistencies between similar files/environments
- [ ] **Missing pieces**: absent tests, missing health checks, no resource limits
- [ ] **Documentation**: stale README, outdated examples
- [ ] **Composite actions** (GitHub Actions repos): step output reference mismatches, undefined shell arrays, unsupported inputs, env var naming consistency. See [references/composite-action-bug-patterns.md](references/composite-action-bug-patterns.md).

## Pitfalls

- **Subagent token limits**: Large repos (500+ files) can exhaust a subagent's iteration budget. Always check the summary quality and supplement with direct review.
- **False positives**: Subagents may report issues that don't exist (e.g., misreading a string literal as a typo). Verify 🔴 findings by reading the file yourself.
- **Version hallucinations**: Subagents frequently get software version numbers wrong — especially for GitHub Actions, npm packages, and API versions. They may claim a version "doesn't exist" when it does, or that a version is "outdated" when it's current. **Always verify version claims against the actual GitHub releases page or upstream source before reporting them.** Use `web_extract` on the repo's releases page to confirm. This is especially common with GitHub Actions — subagents often pattern-match on "v4 is the latest" from old training data and flag v5+, v7+, v9+, v10+ as non-existent when they're actually current.
- **Re-cloning**: Check if `/tmp/review-*` directories already exist before cloning. Subagents from the same session may have already cloned the repo.
- **Scope creep**: If the user asks for a deep security audit, that's a different skill (`security-threat-model`). This skill is for general code review across multiple repos.
- **Too many repos**: If reviewing 5+ repos, prioritize the ones the user mentioned first. You can always review the rest in a follow-up.

## Integration with Other Skills

- **requesting-code-review**: Use for pre-commit verification of YOUR changes to a single repo.
- **codebase-inspection**: Use for LOC metrics and language breakdown of a single repo.
- **security-threat-model**: Use when the user explicitly asks for threat modeling.
- **dogfood**: Use for web application QA testing with browser tools.
