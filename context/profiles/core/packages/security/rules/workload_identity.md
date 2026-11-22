---
paths:
  - ".github/workflows/*"
  - ".github/workflows/*/*"
  - ".github/workflows/*/*/*"
  - "infra/*"
  - "infra/*/*"
  - "infra/*/*/*"
  - "infra/*/*/*/*"
---

# Workload Identity

Zero long-lived service-account keys: no exported key files in CI, in repositories,
or mounted into pods. CI authenticates to the cloud by exchanging its per-run OIDC
token through a federation trust policy pinned to repository and ref; pods
authenticate through a Kubernetes service account bound to a cloud identity that
mints short-lived tokens into the workload.

Discipline that keeps federation tight: trust policies are attribute-scoped (never
issuer-only), each pipeline purpose maps to its own least-privileged identity, and
each workload gets a dedicated service account — a shared identity makes least
privilege and attribution unenforceable. When a how-to suggests downloading a key
file, treat it as a migration target.

- the CI-to-cloud and pod-to-cloud federation mechanics, trust-policy scoping, and per-workload identity rules — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::security@core/reference/workload_identity.md" />
