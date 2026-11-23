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

# Supply-Chain Security

Lock every third-party CI step to an immutable reference and give each job
the narrowest token scope. Sign every container image at build time with
Sigstore Cosign under the CI workflow's OIDC identity — keyless, so no signing key exists to leak. Attach SLSA
provenance attestations so consumers can assert an artifact was built by the
sanctioned pipeline from the declared source. Scan images (Trivy/Grype) before
promotion; fixable critical/high findings fail the release, and accepted risks are
recorded entries with owners, never disabled checks.

Verification runs again at the cluster boundary — a policy controller checks
signature and provenance at admission — and deployment manifests pin images by
immutable digest, because a mutable tag can be repointed after verification and
quietly void the chain.

- keyless OIDC signing, SLSA provenance levels, vulnerability-gate discipline, and digest-pinned admission — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::security@core/reference/supply_chain_signing.md" />
