# GitOps Delivery

GitOps delivery — the build/publish vs deploy separation, topology trade-offs, and the provisioning vs deployment boundary.

GitOps is a control contract, not a tool choice: Git holds the desired state, a
reconciliation controller running in or beside the cluster continuously converges
live state toward it, and no human or pipeline edits the cluster imperatively.
A reviewer reading the diff is reading exactly what will run.

## Separation of concerns

The delivery pipeline has two halves with different credentials and different failure
modes:

- **CI builds and publishes.** On a merged change, CI compiles the artifact, pushes
  it to a registry, and — where versioning is automated — updates the manifest
  repository (a new image tag or digest committed to Git). CI holds registry
  credentials and signing identity; it holds no cluster-admin credentials.
- **GitOps deploys.** A reconciliation controller watches the manifest repository,
  applies desired state, reports drift, and can wait for health checks before
  completing a rollout. It pulls; nothing pushes into the cluster.

The payoff is blast-radius control: a compromised CI pipeline can publish a bad
artifact, but it cannot silently install it — promotion still has to pass through a
Git commit that policy can inspect and gate.

## Topology trade-off

Two cluster-level topologies dominate. Treat the choice as explicit and revisitable,
not dogma.

| | Hub-and-spoke (e.g. Argo CD style) | Controller-only (e.g. Flux style) |
|---|---|---|
| Shape | A central GitOps control plane manages many clusters; applications register through an app-of-apps pattern | Each cluster runs its own small set of controllers; no central server |
| Strengths | Single pane for drift, sync status, and rollbacks; UI aids on-call and audits; access control concentrates in one place | Minimal footprint per cluster; no hub component to operate or secure; cluster autonomy survives hub outages |
| Costs | The hub is itself a workload with credentials to many clusters — it is part of the blast radius | Coordination happens through Git only; no built-in cross-cluster view; multi-tenancy needs more discipline |

Hub-and-spoke fits a platform team operating many clusters as a service.
Controller-only fits flatter organizations or few clusters where a central UI would
be idle weight. Both read the same manifest repository; the topology decision does
not change what a service author writes.

## Provisioning vs deployment boundary

Keep a hard line between infrastructure provisioning and application deployment:

- **IaC provisions** — networks, clusters, databases, IAM, registries. Changes go
  through plan/apply with human review; they are infrequent and high-leverage.
- **GitOps deploys** — workload manifests and releases. Changes are frequent,
  per-service, and gated by merge.

The boundary is the point where a change stops creating capacity and starts being an
application release. Terraform (or its equivalent) stops where the release begins.
A service team adding a database table touches GitOps; a team needing a new database
instance touches IaC. GitOps controllers should not create cloud resources, and IaC
should not ship application revisions.

## Environments and promotion

Per-environment state lives in distinct directories, branches, or overlays; promotion
is a Git operation (bump the target environment's manifest), never a hand-edit of a
live cluster or a re-run of a deploy script. Drift between Git and the cluster is a
defect: the controller reports it, and the fix is a commit, not a manual patch. See
also the deployment-profile model in
<ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::delivery@core/reference/deployment_profiles.md" />.
