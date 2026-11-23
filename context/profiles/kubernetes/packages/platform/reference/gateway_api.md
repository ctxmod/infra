# Gateway API

Gateway API over legacy Ingress — traffic splitting, cross-namespace routing, gRPC/SSE termination, and migration framing with conversion examples.

Gateway API is the modern standard for L7 routing in Kubernetes; the legacy
Ingress resource is frozen. Ingress's limits are structural, not incidental:
routing semantics live in controller-specific annotations, there is no ownership
model for who may reference what across namespaces, traffic splitting is an
afterthought, and status conditions are too thin to debug from. New routing
configuration uses Gateway API; existing Ingress is a migration queue.

Two different things are retiring at different speeds, and conflating them
misjudges the urgency:

- **The Ingress API** (`networking.k8s.io/v1` Ingress) remains served and
  supported; it is feature-frozen, not deprecated for removal.
- **The ingress-nginx controller** (`kubernetes/ingress-nginx`) was retired by
  SIG Network and the Security Response Committee in **March 2026**: no further
  releases, bug fixes, or security patches. Existing installs keep running and
  artifacts stay downloadable, but any newly disclosed vulnerability stays
  open. A cluster still running it carries an unpatchable internet-facing
  component; its migration is a security item, not a cleanup. Find it with
  `kubectl get pods -A -l app.kubernetes.io/name=ingress-nginx`, and check
  managed add-ons (some providers shipped it as a routing add-on) for the
  provider's own support end date.

Gateway API itself is at v1.7.0 (2026-11); its core kinds (GatewayClass,
Gateway, HTTPRoute, GRPCRoute, ReferenceGrant) are in the Standard channel.
Install the CRD bundle version your controller's conformance report names,
rather than the newest release.

## What Gateway API buys

- **Traffic splitting for canaries**: an HTTPRoute attaches multiple backends
  with explicit weights, so a canary is a manifest edit — 5% to the new
  revision, promote by moving the weight — executed by the same GitOps flow as
  everything else.
- **Cross-namespace routing with explicit grants**: a Route in one namespace can
  attach to a Gateway in another only when the Gateway's owner allows it
  (`allowedRoutes`, ReferenceGrants). The platform owns Gateways; service teams
  own their Routes; the trust boundary is enforced by the API, not by convention.
- **Protocol support**: gRPC routes terminate gRPC natively, and streaming
  workloads (SSE, websockets) work without annotation folklore.
- **Debuggable status**: Gateways and Routes report conditions — attached or
  not, and why — so a route that silently serves nothing is visible in the
  resource, not only in a controller's logs.

## Resource model

Three role-separated kinds: **GatewayClass** (the controller implementation,
platform-installed), **Gateway** (the actual listeners — ports, TLS, hostnames;
platform-owned, few in number), and **xRoutes** (HTTPRoute, GRPCRoute, and
siblings; service-owned, many). The separation is the point: platform and
service concerns stop colliding in one object.

## Migration framing

Treat Ingress manifests as legacy with a conversion path, not as a second
supported dialect:

1. Convert mechanically — the SIG Network `ingress2gateway` tool (v1.2.0 as of
   2026-11) translates Ingress plus provider-specific resources to Gateway +
   HTTPRoute form, with an ingress-nginx provider for the common annotations
   and emitters for implementation-specific policy resources. It deliberately
   does not copy annotations it cannot translate, so review its output for
   dropped behavior.
2. Restructure what conversion cannot carry: annotation-based rewrites and
   controller-specific snippets become explicit Route filters or Gateway TLS
   config.
3. Flip traffic per host, verify with the Gateway's status and the dataplane's
   flow view (see ebpf-dataplane.md), then delete the Ingress.

The conversion shape, as a before/after skeleton:

```text
Legacy:  Ingress (host app.example.com, path / → service:app, TLS via annotation)
Modern:  Gateway (listener app.example.com, TLS secret)  ← platform-owned
         HTTPRoute (hostMatch app.example.com → service:app, weights for canary)  ← service-owned
```

Coexistence is transitional. A repository that adds new Ingress manifests after
Gateway API is adopted is reintroducing the legacy pattern, and review should
say so.

## Sources

- https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/ — ingress-nginx retirement, best-effort maintenance until March 2026, Ingress API feature-frozen (accessed 2026-11-22)
- https://kubernetes.io/blog/2026/01/29/ingress-nginx-statement/ — Steering and Security Response Committee statement on the March 2026 retirement (accessed 2026-11-22)
- https://github.com/kubernetes-sigs/gateway-api/releases — Gateway API v1.7.0 (accessed 2026-11-22 via release tags)
- https://github.com/kubernetes-sigs/ingress2gateway — providers (including ingress-nginx), emitters, no annotation copying; v1.2.0 (accessed 2026-11-22)
