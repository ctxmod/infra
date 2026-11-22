---
paths:
  - "infra/*"
  - "infra/*/*"
  - "infra/*/*/*"
  - "infra/*/*/*/*"
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
---

# GitOps

CI compiles and pushes signed artifacts to the registry and records the new version
in Git; a reconciliation controller pulls deployment state from Git into the
cluster. CI holds no cluster credentials, and nothing runs imperative deploys
(`kubectl apply`, manual rollouts) against a live environment — promotion is a Git
commit, and drift between Git and the cluster is a defect to fix with a commit.

The hub-and-spoke vs controller-only topology choice is deliberate, not default:
the hub gives a single drift/rollback pane and concentrates credentials (and blast
radius); controller-only keeps clusters autonomous with no central server. Keep the
provisioning/deployment boundary hard — IaC provisions capacity, GitOps ships
application releases; a change crosses exactly one of the two.

- the build/publish vs deploy split, topology trade-off table, and the provisioning vs deployment boundary — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::delivery@core/reference/gitops.md" />
- the GCP instantiation — registry conventions, progressive delivery as a service, and service-layer IAM separation — `gcp/reference/gcp_delivery.md` (profile `core`)
