# Tool Positioning

IaC tool positioning — HCL Terraform as the baseline, OpenTofu as the OSS-first equivalent, language-native and Kubernetes-native IaC as deliberate alternatives, and the provisioning vs deployment split.

Infrastructure-as-code has four credible shapes. The choice is a real trade-off;
make it once, per estate, and write the rationale down.

## The options

**HCL Terraform — the baseline.** Declarative HCL with the broadest provider
coverage, the largest hiring pool, and plan/apply review semantics everyone
knows. Default choice; it is the position the other three are measured against.

**OpenTofu — the OSS-first equivalent.** A community fork, HCL-compatible, that
keeps the Terraform workflow under a permissive license. Functionally a
drop-in for most estates, though the two now diverge feature by feature in both
directions — check the specific feature, not the brand:

| As of 2026-11-22 | Terraform (1.17.x) | OpenTofu (1.13.x) |
|------------------|--------------------|-------------------|
| Ephemeral values / write-only arguments | 1.10 / 1.11 | 1.11 |
| Client-side state and plan encryption | No (backend encryption only) | Since 1.7 |
| Native testing | `terraform test` (`.tftest.hcl`) | `tofu test` (same file format lineage) |
| Notable recent additions | Terraform Policy GA and variables in provider requirements (1.17.0); Stacks on HCP Terraform | OCI registry sources for modules and providers (1.10), `lifecycle { enabled = ... }` (1.11), experimental `-lint` (1.13) |

A configuration that uses a feature only one side has is no longer portable;
record that as a deliberate lock-in decision. Pick OpenTofu where license posture or community
governance matters; pick one and standardize — running both in one repository
is a non-choice that doubles toolchain surface for nothing.

**Language-native IaC (Pulumi, CDK-style).** Infrastructure in a general-purpose
language: real types, real tests, real abstractions. Earns its cost when infra
logic is genuinely algorithmic (many similar resources, condition-heavy
composition, shared library reuse) and the team already lives in the host
language. Costs: two paradigms to review, and state semantics that differ from
the HCL world.

**Kubernetes-native IaC (Crossplane).** The cluster itself becomes the control
plane for cloud resources — infrastructure declared as CRDs, continuously
reconciled, fitting the GitOps flow already shipping workloads. Earns its cost
when the organization is deeply GitOps/Kubernetes-centric and wants app teams
provisioning service-layer resources through the same PR flow as deployments.
Costs: a continuous-reconciliation failure model (no plan step), an extra
control plane to operate, and composition through the cluster rather than a
static plan.

## The decision rule

Default to the HCL baseline (Terraform or OpenTofu, chosen once). Introduce an
alternative only when its defining strength answers a named requirement — not
for novelty, and not per-team. Mixed estates are legitimate at the layer
boundary (Crossplane for service-layer resources, HCL for the platform layer)
but only when the seam is explicit.

## Provisioning provisions; GitOps deploys

Whatever the tool, IaC stops where application release begins. Terraform (or
Crossplane, or Pulumi) provisions capacity: networks, clusters, databases,
identities, registries. GitOps deploys workload revisions against that capacity.
The boundary keeps plan/apply cadence slow and reviewable while deployment stays
continuous; blending them — a pipeline that runs `terraform apply` per app
release — couples the two blast radii. The delivery-side view of this boundary
lives in the infra profile at
<ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::delivery@core/reference/gitops.md" />.

## Sources

- https://raw.githubusercontent.com/hashicorp/terraform/v1.17/CHANGELOG.md — Terraform 1.17.0 (2026-10-21) features, 1.17.2 (2026-11-11) (accessed 2026-11-22)
- https://raw.githubusercontent.com/opentofu/opentofu/v1.11/CHANGELOG.md — OpenTofu 1.11 ephemerality and `enabled` (accessed 2026-11-22)
- https://raw.githubusercontent.com/opentofu/opentofu/v1.10/CHANGELOG.md — OCI registry module and provider sources (accessed 2026-11-22)
- https://www.infoq.com/news/2026/10/opentofu-1-13-unknown-values/ — OpenTofu 1.13 release coverage (accessed 2026-11-22)
- https://opentofu.org/docs/language/state/encryption/ — OpenTofu state encryption (accessed 2026-11-22)
