# Supply-Chain Signing

Supply-chain signing — keyless OIDC image signing, SLSA provenance, vulnerability gates, and digest-pinned deployment.

An artifact's authority to run should be provable, not assumed. The modern pipeline
signs every artifact at build time, records how it was built, gates deployment on
vulnerability results, and verifies the chain again at the point of admission.

## Keyless signing

Sign container images with Sigstore Cosign using the CI system's OIDC identity
(keyless signing). The signature is bound to the workflow's short-lived identity —
repository, workflow path, ref — instead of a long-lived signing key. Consequences:

- No signing key exists to leak, rotate, or escrow; the trust anchor is the CI
  provider's identity fabric.
- A signature can be verified against an expected identity: "this image was signed
  by the release workflow of this repository at this ref." Signatures from any other
  identity fail verification, even if cryptographically valid.
- Transparency-log entries give the signing event a tamper-evident public record.

The same OIDC identity model replaces stored credentials for registry pushes and
cloud authentication (see workload-identity.md).

## SLSA provenance

A signature says who built the artifact; provenance says how. Generate SLSA
provenance attestations in the build workflow — builder identity, source commit,
build instructions, dependencies — and treat them as first-class release metadata.
Consumers and admission policy can then assert that an artifact was produced by the
declared pipeline from the declared source, closing the gap where a validly-signed
image was built outside the sanctioned process. Aim for the SLSA level that the
platform's hosted builders give by default; document the level actually achieved.

## Vulnerability gates before deploy

Scan images (Trivy, Grype, or equivalent) in the pipeline before promotion, and make
the gate real:

- Critical or fixable-high findings fail the promotion; warnings do not silently
  pass either — they surface in the release record.
- Gate on the delta where the baseline allows: a known-unfixable base-image finding
  is an accepted-risk entry with an owner and a review date, not a disabled check.
- Rescan on a cadence even for artifacts already deployed; a gate at build time
  only protects against shipping new vulnerabilities, not against newly disclosed
  ones in running images.

## Verify at admission, deploy by digest

Verification at build time is necessary and insufficient. Enforce the chain again at
the cluster boundary — a policy controller verifies signature and provenance before
admitting an image. Reference images by immutable digest in deployment manifests; a
mutable tag can be repointed after verification, which quietly voids the whole chain.
A digest pin plus signature verification means "the bytes running are the bytes the
sanctioned pipeline signed."
