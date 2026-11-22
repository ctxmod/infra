# Deployment Profiles

The deployment-profile model — primary (self-hosted) and secondary (managed) profiles, generated pairs that cannot drift, and security enforced in code.

The same service often needs to run in more than one shape: self-hosted on the
platform's own infrastructure, and as a managed or serverless offering. Model these
as named deployment profiles rather than ad-hoc per-environment forks, so the
variation is explicit, bounded, and reviewable.

## Primary profile: self-hosted with generated pairs

The primary profile runs the service on platform infrastructure. Its defining
mechanism is the **generated pair**: for each deployable unit, a generator emits the
runtime unit and its configuration (a systemd unit plus its environment file, a
workload manifest plus its ConfigMap, a service definition plus its ports file)
from one source of truth.

- The two halves of a pair are regenerated together and byte-compared in CI;
  hand-editing one half is a defect the check catches, so a pair cannot drift.
- The generator encodes the invariants once — naming, directory locations, health
  checks, restart policy — and every unit inherits them. A new service gets the
  platform's operational baseline by running the generator, not by copying an
  old unit and hoping.

## Secondary profile: managed and serverless

The secondary profile targets platform-managed runtimes (serverless functions,
managed job runners). The application core is unchanged; the profile changes
packaging and lifecycle — no long-running unit, platform-managed scaling, provider
identity instead of self-managed secrets. Keep secondary profiles genuinely
secondary: they consume the same artifact, and a capability that exists only in
the managed shape (or only self-hosted) is a design smell worth examining.

## Security enforced in code, fail-closed

Hardening is a property of the generated output, never of a checklist applied later:

- Services bind to loopback unless they are explicitly edge-exposed; exposure is a
  declared property with a TLS termination point, not a default.
- mTLS or the platform's equivalent protects service-to-service traffic at the
  boundary; identity comes from the workload identity layer, not shared secrets.
- **Fail closed.** A missing certificate, an unresolvable identity binding, or an
  absent hardening input stops the start with a loud error. There is no silent
  downgrade to an insecure fallback — a config that would yield a world-open bind
  or plaintext edge is a startup failure by construction, because the generator
  refuses to emit it.

The test for "enforced in code" is mechanical: take a fresh checkout, delete a
security input, and run the generator. It must refuse. If it emits something
insecure instead, the enforcement is documentation, and documentation does not
bind at 3 a.m.
