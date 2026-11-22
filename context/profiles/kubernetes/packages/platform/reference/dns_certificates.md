# DNS and Certificates

Automated DNS and certificate issuance — CRD-driven record sync (ExternalDNS-style) and certificate lifecycle (cert-manager-style) composed with the routing layer.

Public-facing routing is fully declarative: a workload declares a hostname and
its certificate, and controllers make both true. No registrar clicks, no manual
CA steps, no expiring-secret emergencies.

## Declared DNS records

An ExternalDNS-style controller watches the cluster's routing resources
(Gateways, Routes, Services with appropriate annotations) and reconciles DNS
records against them: a Route declaring host `app.example.com` causes the
corresponding record to exist, pointed at the Gateway's addresses. DNS becomes
an output of the manifest tree:

- Deleting a route retires its record — no orphaned DNS pointing at dead
  infrastructure.
- Records are reviewed where the routing is reviewed, in the same GitOps diff
  (see manifest-layout.md), rather than changed out-of-band in a registrar
  console.
- Ownership metadata (txt records) keeps multiple DNS sources from fighting
  over the same zone.

## CRD-driven certificates

A cert-manager-style controller owns the certificate lifecycle through
`Certificate` custom resources: request, issue, renew, and store as TLS secrets
in the workload's namespace. ACME (Let's Encrypt) covers public hosts; a
private-CA issuer covers internal mTLS. What the CRD model fixes is the
failure mode of hand-issued certificates — they expire silently. Renewal is a
controller loop with visible status; a certificate that will fail renewal
raises a condition, and alerting watches conditions rather than expiry dates.

Certificate resources live next to the Routes that consume them, and the
Gateway's TLS listeners reference the issued secrets — the three-way binding
(hostname declaration, certificate, route) is all manifest content.

## Composition

The two controllers compose with the routing layer into one flow: declare a
Route with a hostname → DNS record appears → certificate is issued for it →
Gateway terminates TLS with it. A new public endpoint is a reviewed pull
request, and its end-to-end correctness (record exists, cert valid, route
attached) is checkable from resource statuses without touching any external
console.
