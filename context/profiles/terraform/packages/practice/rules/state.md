---
paths:
  - "*.tf"
  - "*/*.tf"
  - "*/*/*.tf"
  - "*/*/*/*.tf"
  - "*/*/*/*/*.tf"
  - "*.tfvars"
  - "*/*.tfvars"
  - "*/*/*.tfvars"
  - "*/*/*/*.tfvars"
  - "*/*/*/*/*.tfvars"
  - "terragrunt.hcl"
  - "*/terragrunt.hcl"
  - "*/*/terragrunt.hcl"
  - "*/*/*/terragrunt.hcl"
  - "*/*/*/*/terragrunt.hcl"
---

# State Layering

Split state by lifecycle and ownership: a platform layer (networks, IAM
baseline, clusters, managed databases) owned by the platform team, and a
service layer (per-service buckets, service accounts, role bindings, topics)
owned by the app teams. The service layer consumes platform outputs; the
platform layer never references service state, so a service apply can never
force a platform plan.

Each layer — and each service within it — gets its own remote state root in a
locked, encrypted backend. State access is least-privilege in both directions:
a service team's credentials touch their root and nothing else. State contains
secrets; treat a state-read grant as a sensitive grant and audit it.

- the platform/service split table, state-root isolation, and backend least-privilege discipline — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::practice@terraform/reference/state_layering.md" />
