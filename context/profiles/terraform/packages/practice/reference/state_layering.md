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
concurrent applies from corrupting state. On the S3 backend, use native S3
locking (`use_lockfile = true`, Terraform 1.10+) rather than a DynamoDB lock
table, whose arguments Terraform's changelog marks for deprecation. Access is least-privilege in both
directions:

- A service team's credentials read and write their own state root, and nothing
  else; platform state is invisible to them, not merely unwritable.
- CI pipelines applying a root hold exactly that root's backend and project
  grants — the same identity-scoping discipline as any other workload
  credential.

Because state can contain secrets, backend access *is* secret access: audit it,
encrypt it at rest, and treat a state-read grant as the sensitive grant it is.

## Keep secrets out of state and plan

Encrypting state contains the exposure; the language now lets configuration
avoid writing the secret at all:

- **Ephemeral resources and values** (Terraform 1.10, OpenTofu 1.11) are read
  anew in each phase and never persisted to state or plan files. Fetch a
  secret from a vault or secret manager with an `ephemeral` block, and mark
  pass-through variables and outputs `ephemeral = true`.
- **Write-only arguments** (Terraform 1.11, OpenTofu 1.11) let a managed
  resource accept a value without storing it: provider arguments ending in
  `_wo` (for example `password_wo`) paired with a `_wo_version` counter that is
  stored and bumped to trigger rotation, because Terraform cannot diff a value
  it never kept.
- **OpenTofu state and plan encryption** (since OpenTofu 1.7) encrypts state
  client-side with a key provider (PBKDF2, AWS/GCP KMS, OpenBao, external
  programs), so the backend operator never sees plaintext. It complements, and
  does not replace, the two mechanisms above.

The rule this supports: a new secret-bearing argument uses the ephemeral or
write-only form when the provider offers one; a plain `password = var.x` on a
provider that has a `_wo` variant is a review finding, because the value lands
in every state snapshot from then on.

## Sources

- https://developer.hashicorp.com/terraform/language/manage-sensitive-data/write-only — write-only arguments, `_wo`/`_wo_version`, Terraform 1.11 (accessed 2026-11-22)
- https://developer.hashicorp.com/terraform/language/manage-sensitive-data/ephemeral — ephemeral resources and values, Terraform 1.10 (accessed 2026-11-22)
- https://raw.githubusercontent.com/hashicorp/terraform/v1.10/CHANGELOG.md — ephemeral values and S3 native locking in 1.10.0, DynamoDB locking slated for deprecation (accessed 2026-11-22)
- https://raw.githubusercontent.com/opentofu/opentofu/v1.11/CHANGELOG.md — ephemeral values and write-only attributes in OpenTofu 1.11.0 (accessed 2026-11-22)
- https://opentofu.org/docs/language/state/encryption/ — OpenTofu state and plan encryption and key providers (accessed 2026-11-22)
