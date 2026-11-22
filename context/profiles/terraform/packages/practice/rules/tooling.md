---
paths:
  - "*.tf"
  - "*/*.tf"
  - "*/*/*.tf"
  - "*/*/*/*.tf"
  - "*/*/*/*/*.tf"
  - "*.tofu"
  - "*/*.tofu"
  - "*/*/*.tofu"
  - "*/*/*/*.tofu"
  - "*/*/*/*/*.tofu"
  - "terragrunt.hcl"
  - "*/terragrunt.hcl"
  - "*/*/terragrunt.hcl"
  - "*/*/*/terragrunt.hcl"
  - "*/*/*/*/terragrunt.hcl"
---

# Tool Positioning

HCL Terraform is the baseline: broadest provider coverage, plan/apply review
semantics, largest talent pool. OpenTofu is the HCL-compatible OSS-first
equivalent — pick one of the two and standardize; running both in one
repository is a non-choice. Language-native IaC (Pulumi, CDK-style) earns its
cost when infra logic is algorithmic and typed; Kubernetes-native IaC
(Crossplane) earns its cost in deeply GitOps-centric estates where app teams
provision service-layer resources through the cluster. Introduce an
alternative only for a named requirement, and keep any mix at an explicit
layer seam.

Whatever the tool, IaC provisions capacity; GitOps deploys releases. A pipeline
that applies Terraform per application release couples two blast radii that
belong apart.

- the four-option trade-off, the decision rule, and the provisioning vs deployment boundary — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::practice@terraform/reference/tool_positioning.md" />
