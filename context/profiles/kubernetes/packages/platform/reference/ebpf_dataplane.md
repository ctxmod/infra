# eBPF Dataplane

eBPF dataplane networking — replacing kube-proxy, identity-based L7 policy, default workload-to-workload isolation, and flow-level observability.

The default Kubernetes dataplane routes traffic with kube-proxy's iptables rules:
per-service rule lists that grow linearly with the cluster, IP-based semantics,
and no view above layer 4. An eBPF dataplane (Cilium is the current-generation
default) replaces that mechanism with programs attached at the kernel's network
hooks — faster path setup at scale, and an identity layer that the security
model builds on.

## Identity-based policy, not IP-based

Under eBPF, every workload carries a security identity derived from its
Kubernetes labels and service account — not from an IP address that changes on
every reschedule. Network policy is written against those identities, and the
dataplane can enforce it at layer 7: allow `GET /healthz` but not `DELETE` to a
path, permit gRPC method families, resolve DNS names in rules. IP-based rules
rot on every pod churn; identity-based rules describe intent and survive it.

## Isolation as the default posture

Write policy default-deny and open explicit paths, rather than open-by-default
with deny patches. The useful phrasing is workload-to-workload: each component
may reach its **designated dependencies and nothing else**. A service whose
purpose is brokering access to a set of tool endpoints, for example, may reach
exactly those endpoints — while its callers reach the broker, never the tools
directly. This converts the network from a flat trust space into explicit,
reviewable edges:

- Egress is policy-managed too; "the cluster cannot reach the internet except
  through declared gateways" is an enforceable statement, not an aspiration.
- Namespace boundaries carry policy, so a compromised workload in one namespace
  does not silently scan another.
- Policy intent lives beside the manifests it protects, in the same GitOps flow
  (see manifest-layout.md).

## Flow-level observability

The same dataplane reports every flow decision — allowed and denied, with L7
detail (HTTP path, gRPC method, DNS query) — without sidecars (Hubble is the
canonical viewer). Two uses earn their keep immediately:

- **Policy verification**: watch real traffic against a new policy before it
  goes from monitor-mode to enforce, and see exactly which flows a tightening
  would break.
- **Investigation**: a latency spike or a connection refusal resolves to
  specific flows between named identities, not to grep across pod logs.

The observability layer is also the migration tool: run the eBPF dataplane in
parallel with kube-proxy during adoption, compare flow behavior, then disable
the legacy path.
