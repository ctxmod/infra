# Manifest Layout

Manifest organization — base/overlays layout, per-environment deltas, and the GitOps sync contract that makes the diff the deployment.

Manifests are deployment code: they need a layout that answers three questions
at a glance — what does a service look like, what differs per environment, and
what ships if this diff merges.

## Base and overlays

The standard shape is a shared base plus per-environment overlays (kustomize or
equivalent composition):

```text
infra/k8s/
  base/
    <service>/          # Deployment, HTTPRoute, NetworkPolicy, Certificate — the shape
  overlays/
    dev/                # references base; patches: image tag, domain, replicas, limits
    prod/               # references base; patches: prod domain, higher limits, tighter policy
```

Rules that keep the shape honest:

- **Base holds the shape, overlays hold the values.** Anything environment-
  specific (image digest, hostname, replica counts, resource limits, feature
  flags) is an overlay patch; if a base contains an environment assumption,
  the next environment inherits it by accident.
- **Patches are small and named.** An overlay that rewrites half the base is a
  different workload wearing the same name — split the base or the service.
- **Policy lives with the workload.** A service's NetworkPolicy sits in its
  base directory, not in a platform-wide policy file nobody attributes; the
  isolation contract (see ebpf-dataplane.md) is part of the service's
  definition.
- **Platform components separate from application manifests.** Gateway
  definitions, cert-manager, DNS controller configuration, and observability
  collectors live under their own roots; a service diff should never be able
  to move the platform beneath other services.

## The GitOps sync contract

The layout is the GitOps source: a reconciliation controller applies
`base + overlays/<env>` per environment, and the repository is the entire
desired state. Consequences that follow from that contract:

- What a reviewer sees in the diff is what ships — nothing between merge and
  cluster modifies it. That makes review meaningful, and it makes rollback a
  revert.
- Environment promotion is a commit that changes the overlay's image reference
  (by digest, per the supply-chain rules in the parent profile) — never an
  in-cluster edit.
- A manual change made directly in the cluster is drift; the controller
  reports it, and the correction lands as a commit or the drift is reverted.
  Which of the two happens is a recorded decision, not an accident of whoever
  notices first.
