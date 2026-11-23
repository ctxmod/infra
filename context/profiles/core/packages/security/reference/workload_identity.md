# Workload Identity Federation

Workload identity federation — zero long-lived keys for CI-to-cloud and pod-to-cloud authentication.

A long-lived service-account key is a permanent credential sitting in a file: it
leaks through repositories, CI logs, and laptop disks, it outlives the rotation
discipline of any team, and its blast radius is bounded only by the account's
grants. Federated identity removes the file. Workloads prove who they are with a
short-lived token minted by a platform they already trust, and the cloud exchanges
it for scoped credentials under a policy.

## The zero-long-lived-keys posture

The rule is absolute: no exported key material for any workload identity, in CI or
at runtime. If a how-to suggests downloading a key file, treat it as a legacy
migration target. Two federation paths cover essentially every case:

## CI to cloud

A CI provider issues an OIDC token per workflow run, carrying the repository,
workflow, ref, and job identity. The cloud IAM side holds a federation
configuration — a trust policy naming the allowed token issuer plus attribute
conditions (which repos, which refs, which workflows) — and maps the token to a
per-pipeline cloud identity. On GCP this is Workload Identity Federation; other
clouds offer the same shape under their own names (OIDC federation roles, managed
identity trust).

Rules that keep the exchange tight:

- Scope the trust policy by attribute, not just by issuer: pin repository and ref
  (`refs/heads/main`, release tags) so a fork or a feature branch cannot mint cloud
  credentials. GCP now requires an attribute condition on GitHub providers.
- Match immutable identifiers, not only names. A repository or owner name can be
  deleted and re-registered by someone else; the numeric `repository_id` and
  `repository_owner_id` claims cannot be reused. GCP's federation guide flags
  name-only conditions as a cybersquatting risk, and GitHub already presents a
  numeric-ID subject form for environment jobs. Pin the IDs (alongside the
  readable name for reviewers), and remember that re-creating a repository
  changes its ID and therefore requires re-creating the trust entry.
- Prefer direct resource access where the cloud offers it: GCP lets IAM roles be
  granted to the federated principal itself, so no intermediate service account
  is impersonated; fall back to impersonation only for services whose federated
  tokens are limited.
- One mapped identity per pipeline purpose; a deploy pipeline gets deploy-scoped
  grants, a scan pipeline gets read-only ones. Never map every pipeline to one
  admin identity.

## Pod to cloud

Inside the cluster, the same exchange runs between the platform's identity system
and cloud IAM: a Kubernetes service account is bound to a cloud IAM identity, and
the platform mints short-lived tokens into the pod — Workload Identity Federation
for GKE (formerly "GKE Workload Identity"), EKS Pod Identity, and Microsoft Entra
Workload ID for AKS are the current instances. On GKE, roles can be granted
directly to a Kubernetes ServiceAccount principal
(`principal://iam.googleapis.com/projects/<number>/locations/global/workloadIdentityPools/<project>.svc.id.goog/subject/ns/<namespace>/sa/<ksa>`)
without binding an IAM service account. Identical namespace and ServiceAccount
names in different clusters of one project share that principal, so keep
namespace names distinct or add conditions. The pod code authenticates through its environment — no secret
mount, no key file.

- Bind one cloud identity per service account, and one service account per
  workload. A shared "app" identity erases attribution and makes least privilege
  unenforceable.
- Grant the cloud identity the minimum the workload needs (a bucket, a database,
  a specific API), not the project.

## Why the posture pays

Short-lived, attribute-scoped credentials shrink leak value to near zero: a stolen
token expires, and the trust policy bounds what any stolen token could reach.
Attribution improves too — audit logs name the workload identity that acted, so an
anomaly maps to a pipeline or a service instead of an anonymous shared account.
When onboarding a new integration, the question is never "where do we store the
key" but "which existing identity federation applies, and what does its policy
need to allow."

## Sources

- https://docs.cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines — required attribute condition for GitHub, direct resource access recommendation, numeric `*_id` claims (accessed 2026-11-22)
- https://docs.cloud.google.com/kubernetes-engine/docs/concepts/workload-identity — Workload Identity Federation for GKE, KSA principal identifiers, identity sameness (accessed 2026-11-22)
- https://docs.github.com/en/actions/reference/security/secure-use — OIDC to cloud providers (accessed 2026-11-22)
