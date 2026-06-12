---
name: systematic-debugging
description: "4-phase root cause debugging: understand bugs before fixing."
version: 1.2.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [debugging, troubleshooting, problem-solving, root-cause, investigation]
    related_skills: [test-driven-development, writing-plans, subagent-driven-development]
---

# Systematic Debugging

## Overview

Random fixes waste time and create new bugs. Quick patches mask underlying issues.

**Core principle:** ALWAYS find root cause before attempting fixes. Symptom fixes are failure.

**Violating the letter of this process is violating the spirit of debugging.**

## The Iron Law

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

If you haven't completed Phase 1, you cannot propose fixes.

## When to Use

Use for ANY technical issue:
- Test failures
- Bugs in production
- Unexpected behavior
- Performance problems
- Build failures
- Integration issues
- Local reverse-proxy / strangler routing problems (client → nginx-strangler → multiple backends)

**Use this ESPECIALLY when:**
- Under time pressure (emergencies make guessing tempting)
- "Just one quick fix" seems obvious
- You've already tried multiple fixes
- Previous fix didn't work
- You don't fully understand the issue

**Don't skip when:**
- Issue seems simple (simple bugs have root causes too)
- You're in a hurry (rushing guarantees rework)
- Someone wants it fixed NOW (systematic is faster than thrashing)

## The Four Phases

You MUST complete each phase before proceeding to the next.

---

## Phase 1: Root Cause Investigation

**BEFORE attempting ANY fix:**

### 1. Read Error Messages Carefully

- Don't skip past errors or warnings
- They often contain the exact solution
- Read stack traces completely
- Note line numbers, file paths, error codes

**Action:** Use `read_file` on the relevant source files. Use `search_files` to find the error string in the codebase.

### 1b. Plan before changing anything

For live infrastructure failures, write a short diagnostic plan before making changes.
- Name the exact failing component.
- Identify the minimum comparison set (usually one healthy node or one working workload).
- Decide what evidence will confirm or reject the hypothesis.
- Only then proceed to a fix.

This prevents cluster debugging from turning into broad manifest thrashing.
### 2. Reproduce Consistently

- Can you trigger it reliably?
- What are the exact steps?
- Does it happen every time?
- If not reproducible → gather more data, don't guess

**Action:** Use the `terminal` tool to run the failing test or trigger the bug:

```bash
# Run specific failing test
pytest tests/test_module.py::test_name -v

# Run with verbose output
pytest tests/test_module.py -v --tb=long
```

### 3. Check Recent Changes

- What changed that could cause this?
- Git diff, recent commits
- New dependencies, config changes

**Action:**

```bash
# Recent commits
git log --oneline -10

# Uncommitted changes
git diff

# Changes in specific file
git log -p --follow src/problematic_file.py | head -100
```

### 4. Gather Evidence in Multi-Component Systems

**WHEN system has multiple components (API → service → database, CI → build → deploy):**

**BEFORE proposing fixes, add diagnostic instrumentation:**

For EACH component boundary:
- Log what data enters the component
- Log what data exits the component
- Verify environment/config propagation
- Check state at each layer

Run once to gather evidence showing WHERE it breaks.
THEN analyze evidence to identify the failing component.
THEN investigate that specific component.

### 5. Trace Data Flow

**WHEN error is deep in the call stack:**

- Where does the bad value originate?
- What called this function with the bad value?
- Keep tracing upstream until you find the source
- Fix at the source, not at the symptom

**Action:** Use `search_files` to trace references:

```python
# Find where the function is called
search_files("function_name(", path="src/", file_glob="*.py")

# Find where the variable is set
search_files("variable_name\\s*=", path="src/", file_glob="*.py")
```

### Phase 1 Completion Checklist

- [ ] Error messages fully read and understood
- [ ] Issue reproduced consistently
- [ ] Recent changes identified and reviewed
- [ ] Evidence gathered (logs, state, data flow)
- [ ] Problem isolated to specific component/code
- [ ] Root cause hypothesis formed

**STOP:** Do not proceed to Phase 2 until you understand WHY it's happening.

---

## Phase 2: Pattern Analysis

**Find the pattern before fixing:**

### 1. Find Working Examples

- Locate similar working code in the same codebase
- What works that's similar to what's broken?

**Action:** Use `search_files` to find comparable patterns:

```python
search_files("similar_pattern", path="src/", file_glob="*.py")
```

### 2. Compare Against References

- If implementing a pattern, read the reference implementation COMPLETELY
- Don't skim — read every line
- Understand the pattern fully before applying

### 3. Identify Differences

- What's different between working and broken?
- List every difference, however small
- Don't assume "that can't matter"

### 4. Understand Dependencies

- What other components does this need?
- What settings, config, environment?
- What assumptions does it make?

---

## Phase 3: Hypothesis and Testing

**Scientific method:**

### 1. Form a Single Hypothesis

- State clearly: "I think X is the root cause because Y"
- Write it down
- Be specific, not vague

### 2. Test Minimally

- Make the SMALLEST possible change to test the hypothesis
- One variable at a time
- Don't fix multiple things at once

### 3. Verify Before Continuing

- Did it work? → Phase 4
- Didn't work? → Form NEW hypothesis
- DON'T add more fixes on top

### 4. When You Don't Know

- Say "I don't understand X"
- Don't pretend to know
- Ask the user for help
- Research more

---

## Phase 4: Implementation

**Fix the root cause, not the symptom:**

### 1. Create Failing Test Case

- Simplest possible reproduction
- Automated test if possible
- MUST have before fixing
- Use the `test-driven-development` skill

### 2. Implement Single Fix

- Address the root cause identified
- ONE change at a time
- No "while I'm here" improvements
- No bundled refactoring

### 3. Verify Fix

```bash
# Run the specific regression test
pytest tests/test_module.py::test_regression -v

# Run full suite — no regressions
pytest tests/ -q
```

### 4. If Fix Doesn't Work — The Rule of Three

- **STOP.**
- Count: How many fixes have you tried?
- If < 3: Return to Phase 1, re-analyze with new information
- **If ≥ 3: STOP and question the architecture (step 5 below)**
- DON'T attempt Fix #4 without architectural discussion

### 5. If 3+ Fixes Failed: Question Architecture

**Pattern indicating an architectural problem:**
- Each fix reveals new shared state/coupling in a different place
- Fixes require "massive refactoring" to implement
- Each fix creates new symptoms elsewhere

**STOP and question fundamentals:**
- Is this pattern fundamentally sound?
- Are we "sticking with it through sheer inertia"?
- Should we refactor the architecture vs. continue fixing symptoms?

**Discuss with the user before attempting more fixes.**

This is NOT a failed hypothesis — this is a wrong architecture.

---

## Red Flags — STOP and Follow Process

If you catch yourself thinking:
- "Quick fix for now, investigate later"
- "Just try changing X and see if it works"
- "Add multiple changes, run tests"
- "Skip the test, I'll manually verify"
- "It's probably X, let me fix that"
- "I don't fully understand but this might work"
- "Pattern says X but I'll adapt it differently"
- "Here are the main problems: [lists fixes without investigation]"
- Proposing solutions before tracing data flow
- **"One more fix attempt" (when already tried 2+)**
- **Each fix reveals a new problem in a different place**

**ALL of these mean: STOP. Return to Phase 1.**

**If 3+ fixes failed:** Question the architecture (Phase 4 step 5).

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Issue is simple, don't need process" | Simple issues have root causes too. Process is fast for simple bugs. |
| "Emergency, no time for process" | Systematic debugging is FASTER than guess-and-check thrashing. |
| "Just try this first, then investigate" | First fix sets the pattern. Do it right from the start. |
| "I'll write test after confirming fix works" | Untested fixes don't stick. Test first proves it. |
| "Multiple fixes at once saves time" | Can't isolate what worked. Causes new bugs. |
| "Reference too long, I'll adapt the pattern" | Partial understanding guarantees bugs. Read it completely. |
| "I see the problem, let me fix it" | Seeing symptoms ≠ understanding root cause. |
| "One more fix attempt" (after 2+ failures) | 3+ failures = architectural problem. Question the pattern, don't fix again. |

## Quick Reference

| Phase | Key Activities | Success Criteria |
|-------|---------------|------------------|
| **1. Root Cause** | Read errors, reproduce, check changes, gather evidence, trace data flow | Understand WHAT and WHY |
| **2. Pattern** | Find working examples, compare, identify differences | Know what's different |
| **3. Hypothesis** | Form theory, test minimally, one variable at a time | Confirmed or new hypothesis |
| **4. Implementation** | Create regression test, fix root cause, verify | Bug resolved, all tests pass |

## Hermes Agent Integration

### Investigation Tools

Use these Hermes tools during Phase 1:

- **`search_files`** — Find error strings, trace function calls, locate patterns
- **`read_file`** — Read source code with line numbers for precise analysis
- **`terminal`** — Run tests, check git history, reproduce bugs
- **`web_search`/`web_extract`** — Research error messages, library docs

### With delegate_task

For complex multi-component debugging, dispatch investigation subagents:

For local routing problems, prefer a quick live-state triage before changing code:
- verify current listeners with `lsof`
- check container logs for upstream variables
- compare compose ports against `.env` examples and docs
- confirm the request path reaches the expected backend before editing routes

```python
delegate_task(
    goal="Investigate why [specific test/behavior] fails",
    context="""
    Follow systematic-debugging skill:
    1. Read the error message carefully
    2. Reproduce the issue
    3. Trace the data flow to find root cause
    4. Report findings — do NOT fix yet

    Error: [paste full error]
    File: [path to failing code]
    Test command: [exact command]
    """,
    toolsets=['terminal', 'file']
)
```

### With test-driven-development

When fixing bugs:
1. Write a test that reproduces the bug (RED)
2. Debug systematically to find root cause
3. Fix the root cause (GREEN)
4. The test proves the fix and prevents regression

## Playwright E2E Race Selectors

When a Playwright test clicks a table/list row and then fails on a missing detail-page heading, inspect the selector contract before changing app code.

Rules of thumb:
- Avoid generic row selectors like `locator('tr').nth(1)` when the table can render loading, empty, or placeholder rows.
- Prefer stable data-row selectors such as `locator('[data-testid^="serviceChainTr-"]').first()` and wait with `await expect(row).toBeVisible()` before clicking.
- If the heading includes dynamic data, use a regex for the stable part instead of exact text.
- Keep the fix scoped to the failing race; do not bundle unrelated flaky tests unless they share the same root cause.

See `references/playwright-e2e-race-selectors.md` for the concrete pattern and snippet.

## Local Reverse Proxy / Strangler Routing

When a client appears to hit the wrong backend in a mixed legacy + NestJS setup:

**First: verify the nginx-strangler container is running.** If it's down, *all* proxied requests fail with 502 — this looks like a routing problem but is an availability problem. Check with `docker ps --filter name=nginx-strangler` and start it if needed.

Then:
1. Verify the client proxy target matches the strangler's host port.
2. Verify the strangler container is publishing the expected host port.
3. Verify the upstream ports in `docker/docker-compose.local.yml` match the live listeners.
4. Read the strangler logs before assuming a route is missing.
5. Only then decide whether the issue is a routing rule, a bad upstream, or a backend 404.

The durable reference for this repo is `references/nginx-strangler-routing.md`.

## NestJS / TypeScript Build Errors

For TS2304, TS1361, and TS2345 errors specific to NestJS controller compilation — missing decorators, `import type` vs value imports, schema misalignment, dead parameters — see `references/nestjs-build-errors.md`. Contains a diagnostic workflow and a case study from this repo.

## Playwright E2E Flaky Table Rows

When a Playwright test clicks a generic table row such as `locator('tr').nth(1)`, verify whether the component renders placeholder rows (`Chargement...`, empty-state rows, skeleton rows) before data arrives. Prefer waiting for a durable data-row selector (`[data-testid^="resourceTr-"]`) and click that locator after `toBeVisible()`. Avoid `toHaveCount(0)` on loaders when the loader may not have mounted yet; wait for the data contract instead.

See `references/playwright-e2e-flaky-table-rows.md` for the condensed debugging pattern.

## NestJS Auth Request-Context Regressions

When auth middleware or guards change how request state is attached, verify every consumer of that state before assuming the new shape is safe.

Rules of thumb:
- If `UserGuard` stores `request.userId` and `request.adminPermissions`, every downstream permission guard/spec should read those flat fields directly and treat missing permissions as `0n`.
- If a shared test helper only constructs headers, seed the request object in the test body; do not pass unsupported properties into the helper.
- If downstream code imports `UserContext`, keep the type source stable during refactors to avoid TS2459/TS2305 import errors.
- Search for all references to the old request contract before patching only one file.

See `references/nestjs-auth-request-context-regression.md` for the concrete reproduction and fix pattern.

## Nixpkgs / Cargo Build Failures

For Cargo `[patch.crates-io]` causing git fetch failures in the Nix sandbox, read-only `Cargo.lock` from the Nix store, and `openssl-src` needing `perl` — see `references/nix-cargo-patch-crates-io.md`.

## Workspace / Package Resolution Failures

When a build or test fails because a workspace package cannot be resolved at import time, suspect eager runtime imports or missing explicit workspace dependencies.
Use `references/workspace-plugin-resolution.md` for the console-repo pattern:
- explicit `workspace:^` deps in the consuming app
- lazy `await import(packageName)` from a fixed allowlist
- await plugin-manager initialization before app setup continues

## Prisma Workspace Generation Races

If parallel workspace tests intermittently fail during `pretest -> prisma generate`, suspect a shared Prisma client output path rather than a code regression. See `references/prisma-workspace-race.md` for the reproduction and the root cause pattern.

## Kubernetes / RKE2 Node Debugging

When a cluster add-on fails with artifact pulls, registry timeouts, DNS errors, or pod startup loops on an RKE2 node, do not jump straight to CNI blame. First separate host reachability from pod reachability, then read the RKE2 journal for the earliest fatal error, then inspect node networking state and kubelet/containerd logs.

Useful split:
- host DNS / egress works, but pod DNS fails: inspect CNI, CoreDNS, NetworkPolicy, and pod-level DNS routing
- add-on pull timeouts plus `lookup github.com on [<cluster-dns-ip>]:53: server misbehaving`: suspect pod DNS or CNI, not the registry itself
- `rke2-canal`/flannel crashes on one IP family while the node is otherwise Ready: verify dual-stack CIDRs, node IPs, and CNI backend compatibility before resetting

Key pitfall: a visible Flux or Helm artifact timeout can be downstream of a node bootstrap error such as an IP-family mismatch (`cluster-cidr` vs `node-ip`) or a partially reconciled CNI setup. If the host can reach the registry but workloads cannot, investigate NetworkPolicy / pod egress / in-pod DNS before planning a cluster reset.

See `references/rke2-flux-node-debugging.md` for the command sequence and heuristics.

## Real-World Impact

From debugging sessions:
- Systematic approach: 15-30 minutes to fix
- Random fixes approach: 2-3 hours of thrashing
- First-time fix rate: 95% vs 40%
- New bugs introduced: Near zero vs common

**No shortcuts. No guessing. Systematic always wins.**