---
paths:
  - "k8s/*"
  - "k8s/*/*"
  - "k8s/*/*/*"
  - "k8s/*/*/*/*"
  - "*/k8s/*"
  - "*/k8s/*/*"
  - "*/k8s/*/*/*"
  - "*/*/k8s/*"
  - "*/*/k8s/*/*"
  - "*/*/*/k8s/*"
  - "kubernetes/*"
  - "kubernetes/*/*"
  - "kubernetes/*/*/*"
  - "kubernetes/*/*/*/*"
  - "*/kubernetes/*"
  - "*/kubernetes/*/*"
  - "*/kubernetes/*/*/*"
  - "*/*/kubernetes/*"
  - "*/*/kubernetes/*/*"
  - "*/*/*/kubernetes/*"
  - "kustomization.yaml"
  - "*/kustomization.yaml"
  - "*/*/kustomization.yaml"
  - "*/*/*/kustomization.yaml"
  - "*/*/*/*/kustomization.yaml"
---

# Kubernetes Networking

Run the eBPF dataplane in place of kube-proxy, and write network policy against
workload identity (labels, service accounts) rather than IPs, with L7 rules
where the protocol allows them. Isolation is the default posture: default-deny,
with each component permitted to reach exactly its designated dependencies —
callers reach a broker, never its backends directly. Flow-level observability
verifies policies in monitor mode before enforcement.

New routing configuration uses Gateway API (Gateway owned by the platform,
Routes owned by services): weighted splitting for canaries, cross-namespace
attach only via explicit grants, native gRPC/SSE termination. Ingress is legacy
— convert it (`ingress2gateway`), never add new instances; a cluster still
running the ingress-nginx controller, retired in March 2026 with no further
security patches, treats that migration as a security item.

- dataplane replacement, identity-based policy, default-deny isolation, and flow observability — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::platform@kubernetes/reference/ebpf_dataplane.md" />
- Gateway API capabilities, resource model, and the Ingress conversion path — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::platform@kubernetes/reference/gateway_api.md" />
