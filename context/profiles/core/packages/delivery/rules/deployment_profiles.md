---
paths:
  - "infra/*"
  - "infra/*/*"
  - "infra/*/*/*"
  - "infra/*/*/*/*"
---

# Deployment Profiles

Model deployment shapes as named profiles instead of per-environment forks. The
primary (self-hosted) profile generates unit-and-config pairs from one source of
truth; the halves are regenerated together and byte-compared in CI, so hand-editing
one half is a caught defect, and a new service inherits the platform's operational
baseline by running the generator. Secondary (managed/serverless) profiles consume
the same artifact with different packaging; a capability existing in only one shape
is a design smell.

Security is a property of the generated output: loopback-only binds unless
edge-exposure is declared, mTLS at the service boundary, and fail-closed startup —
a missing certificate or unresolvable identity binding stops the start rather than
downgrading silently. The mechanical test: delete a security input and run the
generator; it must refuse, not emit something insecure.

- primary/secondary profile model, generated-pair drift prevention, and the fail-closed security posture — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::delivery@core/reference/deployment_profiles.md" />
