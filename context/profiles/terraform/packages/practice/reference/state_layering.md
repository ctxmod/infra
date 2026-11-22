# State Layering

Terraform state layering — platform vs service ownership split, separate state roots, remote state with locking, and least-privilege state access.

Terraform state is the most sensitive artifact in the IaC estate: it is the
authoritative record of what exists, it frequently contains secret values in
plain text, and whoever can write it can reshape the infrastructure. Layer it by
lifecycle and ownership so that no single state holds everything and no single
team can touch everything.

## Platform layer vs service layer

Split by who owns the change and how often it happens:

| | Platform layer | Service layer |
|---|---|---|
| Owns | Shared VPC/network, IAM baseline, clusters, managed databases, registries | Per-service buckets, service accounts, IAM role bindings, Pub/Sub topics, databases schemas' infrastructure |
| Change cadence | Slow, deliberate, platform-team-reviewed | Fast, per-service, app-team-reviewed |
| Blast radius | Everything | One service |

Two properties make the split real. First, **dependency direction**: the service
layer consumes platform outputs (network ids, cluster endpoints) as inputs;
the platform layer never references service state, so a service's apply can
never force a platform plan. Second, **ownership**: a service team applies its
own layer without platform approval, and platform changes never wait on service
review queues.

## Separate state roots

Each layer, and within the service layer each service, gets its own state root.
Grouping tooling (Terragrunt-style) or plain per-root backend configuration both
work; the requirement is mechanical isolation — a plan for one root can neither
read nor write another root's resources. Within a root, keep environments as
separate state as well (per-environment directories or workspaces applied from
the same configuration); a dev mistake must not be able to surface in prod's
dependency graph.

## Remote state, locking, least privilege

State lives remotely (a versioned, encrypted backend — GCS/S3-class), never in
the repository, and never on individual machines. Backend locking prevents
concurrent applies from corrupting state. Access is least-privilege in both
directions:

- A service team's credentials read and write their own state root, and nothing
  else; platform state is invisible to them, not merely unwritable.
- CI pipelines applying a root hold exactly that root's backend and project
  grants — the same identity-scoping discipline as any other workload
  credential.

Because state can contain secrets, backend access *is* secret access: audit it,
encrypt it at rest, and treat a state-read grant as the sensitive grant it is.
