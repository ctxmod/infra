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
to stay compatible. Pin Kubernetes versions deliberately (match dev → prod),
keep manifests and controllers compatible with the supported window, and let
node-pool upgrades roll through non-prod first. Version drift between
environments turns every incident into two investigations; it is cheaper to
hold one version.
