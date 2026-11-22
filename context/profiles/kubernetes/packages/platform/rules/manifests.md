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
  - "overlays/*"
  - "overlays/*/*"
  - "overlays/*/*/*"
  - "overlays/*/*/*/*"
  - "*/overlays/*"
  - "*/overlays/*/*"
  - "*/overlays/*/*/*"
  - "*/*/overlays/*"
  - "*/*/overlays/*/*"
  - "*/*/*/overlays/*"
---

# Manifest Layout

Organize manifests as a shared base plus per-environment overlays: the base
holds the workload's shape (Deployment, Route, NetworkPolicy, Certificate),
overlays hold only values (image digest, domain, replicas, limits). A patch
rewriting half the base is a different workload — split it instead. Policy
lives beside the workload it protects, and platform components (Gateways,
controllers) live in separate roots a service diff cannot move.

The layout is the GitOps source: the controller applies base plus the
environment's overlay, so the diff a reviewer approves is exactly what ships.
Promotion is a commit that bumps the overlay's image reference by digest;
in-cluster manual edits are drift to revert or commit deliberately.

- base/overlays rules and the GitOps sync contract, drift, and promotion mechanics — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::platform@kubernetes/reference/manifest_layout.md" />
