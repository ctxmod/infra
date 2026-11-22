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
---

# Hardened Modules

Stand up infrastructure by consuming internal modules that ship with security
already on: private access endpoints by default, workload identity wired in at
creation, least-privilege IAM generated from the resource type. Opting **out**
of a default requires a named flag reviewers see; a module that requires opt-in
for its own hardening is not hardened.

Consume modules by pinned semver version — never by branch or cross-root local
path — so platform upgrades show as reviewable per-service version diffs.
Deprecations are loud: deprecated attributes keep their names for the window,
removed names are reserved, and a removed input a consumer still sets fails the
plan naming the field.

- security-default design, versioning/consumption contract, and reserved-field deprecation mechanics — <ctxmod_link path="ctx:github.com/ctxmod/infra/context::shared::practice@terraform/reference/modules.md" />
