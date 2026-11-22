# Hardened Modules

Hardened versioned internal Terraform modules — security defaults baked in, semver consumption, and reserved fields on deprecation.

Internal modules are the mechanism that makes the secure path the default path.
A service that stands up a bucket, a service account, or a cluster by consuming
the platform module inherits the platform's security posture without having to
know it; a service that writes raw resources re-decides every security question
by hand, and eventually decides one wrong.

## Security defaults baked in

Each internal module ships with the hardening already on:

- **Private access by default** — resources are reachable through private
  endpoints (Private Google Access-style paths) rather than public addresses;
  public exposure is an explicit opt-in flag that reviewers see and question.
- **Workload identity wired in** — modules that create compute create the
  federated identity binding alongside it, so a workload is born with its
  short-lived cloud identity instead of being handed keys later.
- **Least-privilege IAM** — generated role bindings grant the minimum the
  resource type implies, never project-wide wildcards.

The design rule is asymmetric on purpose: opting **out** of a security default
requires a named, visible flag; opting **in** is never required. When consumers
keep overriding a default, that is a signal to re-examine the default — not to
add another flag.

## Versioning and consumption

Modules are versioned with semver and consumed by exact or caret-pinned version,
never by branch or by local path across service boundaries. A major version bump
is the contract that behavior changed incompatibly; changelogs say what. Pinning
is what makes platform upgrades reviewable — a diff shows precisely which
services moved to which module version.

## Deprecation: reserved fields, loud removal

Deprecate attributes rather than silently repurposing them:

- Mark the attribute deprecated and keep its position in the interface for the
  deprecation window, so existing call sites keep planning cleanly.
- Reserve the name (and, for provider schemas, the field number) after removal
  so a future field can never silently rebind to an old consumer's meaning.
- Remove loudly: a removed input that a consumer still sets must fail the plan
  with an error naming the field, never fall through to a default. The failure
  is the migration signal; silence is how configuration rot becomes an outage.

For the state-layer consequences of module boundaries — which layer owns which
root — see state-layering.md.
