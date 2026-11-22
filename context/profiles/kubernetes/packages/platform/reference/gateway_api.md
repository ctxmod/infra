# Gateway API

Gateway API over legacy Ingress — traffic splitting, cross-namespace routing, gRPC/SSE termination, and migration framing with conversion examples.

Gateway API is the modern standard for L7 routing in Kubernetes; the legacy
Ingress resource is frozen. Ingress's limits are structural, not incidental:
routing semantics live in controller-specific annotations, there is no ownership
model for who may reference what across namespaces, traffic splitting is an
afterthought, and status conditions are too thin to debug from. New routing
configuration uses Gateway API; existing Ingress is a migration queue.

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

1. Convert mechanically — the `ingress2gateway` tool translates most Ingress
   manifests to Gateway + HTTPRoute form.
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
