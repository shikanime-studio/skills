# Subagent Review Pitfalls

## False Positives and Fabrication

Subagents can report issues that don't exist. They may also fabricate specific details (like typos or version numbers) that sound plausible but are false. Always verify critical (🔴) findings by reading the source file directly.

**Example from session (2026-06-12):**
- Subagent reported a "sapling typo" in `land/action.yaml:43` claiming the string was misspelled.
- Direct file read confirmed all occurrences were correct — no typo existed.
- Root cause: terminal output rendering or subagent misread.

**Example from session (2026-06-13):**
- Subagent claimed Kustomize Component API `v1alpha1` should be migrated to `v1beta1`.
- Upstream KEP 1802 shows Components are still in alpha — `v1beta1` graduation is pending.
- Root cause: subagent assumed all `v1alpha1` APIs follow the same migration timeline as Kubernetes core APIs.

**Example from session (2026-06-13):**
- Subagent claimed `actions/github-script@v9`, `actions/cache@v5`, `actions/upload-artifact@v7`, `actions/download-artifact@v7`, `actions/stale@v10`, and `ghaction-import-gpg@v7` all "don't exist."
- All 6 versions **exist and are current** (released 2025-2026).
- Root cause: subagent pattern-matched on "v4 is the latest" from old training data and flagged all newer versions as non-existent.
- **Mitigation:** Always verify version claims against the actual GitHub releases page using `web_extract`. Never report version issues without independent verification.

## Token/Iteration Limits

Subagents have a max iteration count (typically 50). Large repos can exhaust this before the subagent finishes its summary.

**Example from session (2026-06-12):**
- Reviewing `shikanime-studio/devlib` (100+ Nix files) caused the subagent to hit max_iterations and return an error instead of a summary.
- Mitigation: Check `status` and `exit_reason` in the result. If `exit_reason == "max_iterations"`, clone the repo yourself and do a direct review.

## Version Hallucinations

Subagents frequently hallucinate version numbers for software packages, GitHub Actions, and API versions. They may claim a version "doesn't exist" when it does, or that it's "outdated" when it's current.

**Example from session (2026-06-12):**
A subagent reviewing `shikanime-studio/actions` claimed the following versions "don't exist":
- `actions/github-script@v9` — **actually exists**, v9.0.0 released Apr 9, 2026
- `actions/cache@v5` — **actually exists**, v5.0.5 released Apr 13, 2026
- `actions/upload-artifact@v7` — **actually exists**, v7.0.1 released Apr 10, 2026
- `actions/download-artifact@v7` — **actually exists**, v7.0.0 released Dec 12, 2025
- `actions/stale@v10` — **actually exists**, v10.3.0 released May 20, 2026
- `crazy-max/ghaction-import-gpg@v7` — **actually exists**, v7.0.0 released Mar 2, 2026

All 6 claims were **wrong**. The subagent pattern-matched on "v4 is the latest" from old training data.

**Root cause:** Subagents anchor on common training data patterns. When GitHub Actions major versions advance beyond what was common in training (v4 for most actions), subagents assume newer versions are hallucinations. Always verify with `web_extract` on the actual releases page.

**Mitigation:** For any version-related claim (especially "X doesn't exist" or "X is outdated"), verify against the actual GitHub releases page using `web_extract`. Never report version issues to the user without independent verification.

## Understand Before You Change

Some configurations that look wrong have valid reasons. Always investigate *why* something is set a certain way before recommending a change.

**Examples from session (2026-06-13):**

1. **Privileged Pod Security Admission** — `clusters/*/base/ns.yaml` sets `pod-security.kubernetes.io/enforce: privileged`. This looks like a security anti-pattern, but `catbox` genuinely needs `privileged: true` with `hostPath` mounts (Docker-in-Docker style). Changing to `baseline` would break catbox. The correct fix is per-app namespaces, not a blanket change.

2. **`ubuntu-slim` runner** — All workflows use `runs-on: ubuntu-slim` which is not a standard GitHub-hosted runner label. This looked like a mistake, but it's a deliberate cost optimization — `ubuntu-slim` is a smaller/cheaper self-hosted runner used for lightweight tasks (commands, triage, cleanup), while `ubuntu-latest` is reserved for heavy Nix builds. This pattern is consistent across `shikanime/manifests`, `shikanime-studio/actions`, and `shikanime-studio/devlib`.

3. **VPA vs PDBs** — The manifests repo uses VPA (Vertical Pod Autoscaler) without Pod Disruption Budgets. This might look like a missing safeguard, but PDBs conflict with VPA's automatic resource adjustment. VPA is the canonical approach for this setup.

4. **Kustomize Component API** — `v1alpha1` for Components is not deprecated; it's still the current upstream API. Don't recommend migrating to `v1beta1` until upstream does.

**Mitigation:** When you find something that looks wrong, ask "what workloads depend on this?" and "is there a canonical reason for this setting?" before flagging it. Check upstream documentation for deprecation status before recommending API migrations.

## Configuration That Looks Wrong But Isn't

Some configurations that look like mistakes are intentional optimizations or constraints. Investigate before flagging.

**Examples from session (2026-06-13):**

1. **`ubuntu-slim` runner** — Not a standard GitHub-hosted runner label. Used deliberately across shikanime repos as a cost optimization for lightweight tasks (commands, triage, cleanup). `ubuntu-latest` is reserved for heavy Nix builds. This is consistent and intentional.

2. **Privileged Pod Security Admission** — `pod-security.kubernetes.io/enforce: privileged` looks like a security anti-pattern, but some workloads (like catbox with Docker-in-Docker) genuinely need it. Check which workloads require it before recommending a change.

3. **VPA without PDBs** — Pod Disruption Budgets conflict with VPA's automatic resource adjustment. Using VPA alone is a valid pattern.

4. **Kustomize Component API `v1alpha1`** — Still the current upstream API. Don't recommend migrating to `v1beta1` until upstream does.

## Subtle Resource Reference Bugs

Some bugs only manifest when you trace references across files. Subagents often miss these because they review files in isolation.

**Example from session (2026-06-13):**
- `apps/jellyfin/base/netpol.yaml:30` references port `https`
- `apps/jellyfin/base/sts.yaml` only exposes port `http` (8096)
- The `https` port (8920) is only added by the TLS component overlay
- The base netpol would never match traffic without the TLS component
- This cross-file reference bug was caught by direct review, not by the subagent

**Mitigation:** When reviewing NetworkPolicies, trace port names back to the actual container ports in the base StatefulSet/Deployment. Don't assume the netpol author verified the reference.

## Verification Protocol

For every 🔴 Critical finding from a subagent:
1. Read the exact file and line referenced
2. Confirm the issue exists in the actual source
3. Only then report it to the user

For 🟡 Medium findings: spot-check at least 2-3 findings. If they're accurate, the rest are likely reliable.

For 🟢 Low findings: generally trustworthy but may include style preferences.

## User Follow-Up Pattern

When the user asks follow-up questions about specific findings (e.g., "have you checked X?", "what about Y?"), they're often testing whether you:
1. Actually verified your claims or just trusted the subagent
2. Understand the *reason* behind configurations
3. Know when something is intentionally different from what looks "standard"

Treat follow-up questions as signals to do deeper research — check upstream docs, verify against primary sources, and confirm assumptions before answering.
