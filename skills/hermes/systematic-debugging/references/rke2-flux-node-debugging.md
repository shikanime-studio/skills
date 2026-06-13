# RKE2 / Flux node debugging notes

Use this when a Kubernetes add-on reports registry/artifact pull failures, DNS
errors, or pod startup problems on an RKE2 node.

## Case study: manash dual-stack Canal failure

Observed symptom cluster:

- Flux operator reported `ArtifactFailed` while fetching
  `oci://ghcr.io/controlplaneio-fluxcd/flux-operator-manifests:latest`
- node host DNS/egress was reachable, but pod DNS was not
- `rke2-canal` was CrashLoopBackOff with flannel v6 errors
- CoreDNS logged UDP timeouts to `1.1.1.1` / `8.8.8.8`
- Flux source-controller later failed with
  `lookup github.com on [2001:cafe:43::a]:53: server misbehaving`

Root cause pattern:

- host-side connectivity was fine
- pod networking was broken in the dual-stack IPv6 path
- the visible Flux pull failure was downstream of the cluster DNS/CNI issue

Practical lesson:

- if the host can resolve `ghcr.io` or `github.com`, but pods cannot resolve
  external names through CoreDNS, check Canal/Flannel and dual-stack CIDRs
  before touching Flux config
- if `rke2-canal` is failing on one IP family while the node is otherwise Ready,
  verify the CNI backend against the configured pod/service CIDRs before
  resetting

## Diagnostic sequence

1. Separate host reachability from pod reachability.
   - Test the registry from the node host first.
   - Then inspect cluster pods, NetworkPolicies, and the failing controller's
     pod logs.

2. Read the RKE2 server journal for the real bootstrap error.
   - Search for `cni`, `canal`, `flannel`, `calico`, `cilium`, `containerd`,
     `runtime core not ready`, `pod sandbox`, and `cluster-cidr`.
   - The earliest fatal error is often the root cause, while later add-on
     failures are symptoms.

3. Check for IP-family mismatches.
   - A fatal error like
     `cluster-cidr ... and node-ip ... must share the same IP version` means the
     node is starting with incompatible IPv4/IPv6 settings.
   - Verify the configured node IPs, the cluster CIDRs, and whether the cluster
     is meant to be dual-stack.
   - For dual-stack Canal/Flannel failures, confirm that the configured IPv6 pod
     CIDR, service CIDR, and flannel backend all line up with the node address
     family.

4. Inspect node networking state.
   - `ip -br link`, `ip -br addr`, `ip route`, `ip -6 route`.
   - Look for pod CIDR routes, CNI bridge/overlay devices, and stale interfaces
     that indicate an incomplete reconciliation.

5. Use kubelet/containerd logs to distinguish CNI vs higher-level failures.
   - If kubelet shows sandbox/network-plugin errors, focus on CNI.
   - If the node has healthy CNI interfaces and no sandbox errors, artifact
     pulls or NetworkPolicy/pod egress restrictions are more likely than a
     broken CNI plugin.

## Heuristic

If the host can reach the registry but the workload cannot, prefer
investigating:

- pod egress filtering
- NetworkPolicy
- proxy / DNS settings inside pods
- controller-specific configuration

before resetting the cluster.

## Common trap

Do not assume a registry timeout automatically means CNI is broken. In RKE2,
add-on reconciliation can keep retrying while the node is actually failing for a
separate reason, and the first visible symptom may be far downstream of the true
fault.

## Useful commands

```bash
journalctl --no-pager -u rke2-server | egrep -i
  'cni|canal|flannel|calico|cilium|containerd|runtime core not ready|pod sandbox|cluster-cidr'
ip -br link
ip -br addr
ip route
ip -6 route
```
