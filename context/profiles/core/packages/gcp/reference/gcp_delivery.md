# GCP Delivery

GCP delivery — Artifact Registry, progressive delivery services, and service-layer IAM separation. The same shape maps to equivalent services on other clouds.

GCP delivery wiring instantiates the patterns in this profile's delivery references
with concrete services. The shape — immutable registry, delegated release
sequencing, per-service identities — transfers to other clouds; the names below are
the GCP instances.

## Artifact Registry

Container images and language packages live in Artifact Registry, one repository
per service (or per clearly bounded group), regionally located near the cluster.
Conventions that keep it trustworthy:

- Deploy by digest; tags are labels for humans, not deployment references (see
  supply-chain-signing.md).
- CI identities push; runtime identities pull; the two are distinct identities
  with distinct grants.
- Retention policy prunes old images on a schedule — Artifact Registry cleanup
  policies, with keep rules for released digests; the registry is a cache of
  build output, not an archive system.

## Progressive delivery as a service

Release sequencing — canary steps, bake times, approval gates, rollback — belongs
to a delivery service, not to a hand-run script. Cloud Deploy is the GCP-native
option: it defines named pipelines (dev → staging → prod with explicit promotion
steps and approvals) and drives progressive rollout strategies against the cluster,
rolling back on failed health checks. A GitOps controller can carry the same
responsibilities for simpler pipelines; the requirement is that the strategy —
canary weights, bake durations, rollback triggers — is declared in versioned
configuration, so a release behaves the same way at 2 p.m. and at 2 a.m.

Service teams declare *which* strategy and *which* gates; the delivery service
executes. Nobody upgrades a production canary by typing commands.

## Service-layer IAM separation

IAM mirrors the ownership split (see the Terraform state-layering reference for the
full platform/service split):

- **Platform layer** owns shared infrastructure identities: the cluster's node
  pools, shared service agents, registry-level CI pushers. Managed by the platform
  team's IaC.
- **Service layer** gives every service its own Kubernetes service account, its own
  Workload Identity Federation for GKE principal, and grants to exactly the
  resources it owns. Grant roles directly to the KSA principal where the target
  service supports federated tokens; bind an IAM service account for
  impersonation only where it does not (the workload-identity reference owns the
  principal form). A service's
  identity can read its own buckets and databases; it cannot see another service's.
- CI identities are separate again from runtime identities — a pipeline that can
  deploy is not an identity that runs in production.

The practical test: `iam-policy` diffs should be small, reviewable, and attributable
to one service's change. A project where every identity can reach everything fails
it, and so does any audit question of the form "what can this service actually do?"

## Relation to the rest of the profile

Federation details live in workload-identity.md, signing and provenance in
supply-chain-signing.md, and the topology choice (which the delivery service
serves) in gitops.md. This reference adds only the GCP-specific service names and
the IAM separation discipline.

## Sources

- https://docs.cloud.google.com/kubernetes-engine/docs/concepts/workload-identity — Workload Identity Federation for GKE, direct KSA principal grants and their limitations (accessed 2026-11-22)
- https://docs.cloud.google.com/artifact-registry/docs/repositories/cleanup-policy — cleanup policies with delete and keep rules (accessed 2026-11-22)
- https://docs.cloud.google.com/deploy/docs/overview — Cloud Deploy delivery pipelines, promotion, and rollout strategies (accessed 2026-11-22)
