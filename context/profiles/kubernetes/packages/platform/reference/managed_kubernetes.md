# Managed Kubernetes

Managed Kubernetes notes — dataplane enablement, autoscaling delegated to the cluster layer, and upgrade/version posture.

Managed offerings (GKE is the reference instance; EKS/AKS carry the same shape)
own the control plane and a growing share of the platform plumbing. The managed
model changes where your decisions go, not whether you make them.

## Dataplane enablement

Managed offerings increasingly ship their own eBPF dataplane mode — GKE
Dataplane V2 is the example — which is the managed path to the identity-based
policy model (see ebpf-dataplane.md). Enable it at cluster creation where
possible: it is a per-cluster property, and retrofitting means a disruptive
dataplane cutover. On clusters where the provider's mode is unavailable or too
young, install the eBPF layer yourself and disable the legacy kube-proxy path
once flows match. Either way, one dataplane owns routing; two active paths
make policy behavior ambiguous.

Feature gates are also cluster-creation decisions in managed land (Gateway API
support, workload identity federation bindings). Decide the platform baseline
once per environment, and record it with the cluster's IaC.

## Autoscaling is the cluster layer's job

Do not script capacity; declare demand and let the platform layers translate:

- **Service teams** declare resource requests/limits and, where useful, pod
  autoscalers (HPA on metrics, or event-driven scaling). That is the workload's
  whole contribution to scaling.
- **The cluster layer** converts declared demand into nodes: node auto-scaling,
  node pools shaped by workload class (spot/preemptible for tolerant work,
  stable pools for latency-sensitive work), and consolidation. A service that
  "scales" by reserving capacity outside its manifests is bypassing the layer
  and will collide with everyone else's bin-packing.

The division holds at failure time too: a node dying is the cluster layer's
event, and the service's only contract is that its pod spec is schedulable and
its disruption budget honest.

## Upgrades and versions

Managed control planes upgrade on the provider's schedule; the estate's job is
to stay compatible. Upstream maintains the three newest minor releases with
patches for roughly fourteen months each; as of 2026-11 that is 1.37 (newest,
1.37.2), 1.36 and 1.35, with 1.34 at end of life since 2026-10-27. A cluster
two minors behind upstream is one provider deadline from a forced upgrade. Pin Kubernetes versions deliberately (match dev → prod),
keep manifests and controllers compatible with the supported window, and let
node-pool upgrades roll through non-prod first. Version drift between
environments turns every incident into two investigations; it is cheaper to
hold one version.

## Platform primitives to assume on a supported version

Every supported minor (1.35+) has these as stable APIs, so new manifests use
them instead of the workarounds that predate them:

| Primitive | Stable since | Replaces |
|-----------|--------------|----------|
| Pod Security Admission (`pod-security.kubernetes.io/enforce: restricted` namespace labels) | 1.25 | PodSecurityPolicy (removed in 1.25) |
| ValidatingAdmissionPolicy (in-process CEL validation) | 1.30 | Validating webhooks for simple policy |
| Native sidecar containers (`initContainers` with `restartPolicy: Always`) | 1.33 | Sidecars as regular containers with ordering hacks; Jobs that never finish because a proxy keeps running |
| In-place pod resize (`resize` subresource, `resizePolicy`) | 1.35 | Recreating pods to change CPU/memory; VPA's in-place mode was still beta at that graduation |
| MutatingAdmissionPolicy (in-process CEL mutation) | 1.36 | Mutating webhooks for simple defaults |

Prefer the in-process CEL policies over webhooks wherever the rule fits in an
expression: no extra deployment on the admission path, no webhook outage that
blocks every apply. Signature and provenance verification (core's
supply-chain reference) still needs a policy controller.

## Sources

- https://kubernetes.io/releases/ — maintained minor releases and end-of-life dates (accessed 2026-11-22; patch versions from `kubernetes/kubernetes` tags)
- https://kubernetes.io/docs/concepts/security/pod-security-admission/ — Pod Security Admission (accessed 2026-11-22)
- https://kubernetes.io/blog/2024/04/24/validating-admission-policy-ga/ — ValidatingAdmissionPolicy GA in 1.30 (accessed 2026-11-22)
- https://kubernetes.io/blog/2025/04/23/kubernetes-v1-33-release/ — sidecar containers stable in 1.33 (accessed 2026-11-22)
- https://kubernetes.io/blog/2025/12/19/kubernetes-v1-35-in-place-pod-resize-ga — in-place pod resize stable in 1.35; VPA in-place mode beta (accessed 2026-11-22)
- https://kubernetes.io/docs/reference/access-authn-authz/mutating-admission-policy/ — MutatingAdmissionPolicy, `admissionregistration.k8s.io/v1` in 1.36 (accessed 2026-11-22)
