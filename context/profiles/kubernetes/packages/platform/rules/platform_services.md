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

# Platform Services

DNS and certificates are declarative: a Route's hostname drives an
ExternalDNS-style record sync, and a Certificate resource drives
cert-manager-style issuance and renewal — a new public endpoint is a reviewed
pull request, never registrar or CA console work. Alerts watch certificate and
issuer conditions, not expiry dates.

On managed Kubernetes, enable the provider's eBPF dataplane mode at cluster
creation and decide feature gates once per environment, recorded in the
cluster's IaC. Scaling divides by layer: service teams declare
requests/limits and pod autoscalers; the cluster layer turns demand into
nodes. Nobody reserves capacity outside manifests, and nobody scripts control
plane work the provider already owns.

- CRD-driven DNS and certificate lifecycle and their composition with routing — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::platform@kubernetes/reference/dns_certificates.md" />
- dataplane enablement, autoscaling delegation, and upgrade/version posture — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::platform@kubernetes/reference/managed_kubernetes.md" />
