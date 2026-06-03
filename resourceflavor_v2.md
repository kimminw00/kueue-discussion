This document proposes a consistent strategy for naming and using Kueue `ResourceFlavor` resources in heterogeneous accelerator clusters.

The goal is to make accelerated compute resources easier to manage across different hardware classes, interconnect topologies, network fabrics, CPU architectures, and availability tiers. With a consistent naming convention and label model, platform teams can make scheduling behavior more predictable while giving users a simple way to express hardware requirements.

## ResourceFlavor naming convention

Use the following naming pattern for accelerator-backed `ResourceFlavor`s:

```text
<accelerator-vendor>-<accelerator-name>-<interconnect-topology>-<network-fabric>-<cpu-arch>-<availability>
```

This name is intentionally descriptive. It encodes the major scheduling-relevant properties of the node pool directly into the `ResourceFlavor` name, while the actual scheduling match is still driven by node labels.

The `accelerator-vendor` component is included intentionally. In multi-cluster and hybrid cluster environments, accelerator capacity may come from different vendors, providers, regions, or infrastructure domains. Including the vendor name at the beginning of the `ResourceFlavor` makes it easier to distinguish heterogeneous accelerator pools without inspecting extended resource names, node labels, or provider-specific inventory metadata.

The `interconnect-topology` component describes the effective intra-node accelerator interconnect structure visible to workloads. It intentionally does not expose implementation-specific details such as NVLink mesh, NVSwitch, NVLink bridge, PCIe switch layout, or other physical wiring models.

The notation follows this format:

```text
<interconnect-group-count>x<accelerators-per-interconnect-group>
```

For example:

```text
2x2
```

means:

```text
2 interconnect groups
2 accelerators per interconnect group
```

Similarly:

```text
2x4
```

means:

```text
2 interconnect groups
4 accelerators per interconnect group
```

And:

```text
1x8
```

means:

```text
1 interconnect group
8 accelerators per interconnect group
```

The scheduling contract should describe the effective accelerator communication structure that matters to workloads, not the exact hardware implementation.

## ResourceFlavor naming table

| Component | Examples |
|---|---|
| `accelerator-vendor` | `nvidia`, `amd`, `google`, `aws`, `intel` |
| `accelerator-name` | `h100-sxm`, `a100-80gb-pcie`, `h200-nvl` |
| `interconnect-topology` | `2x4`, `1x4`, `2x2`, `1x8` |
| `network-fabric` | `ib-ndr-400g`, `ib-hdr-200g`, `eth-400g`, `eth-100g` |
| `cpu-arch` | `amd64`, `arm64` |
| `availability` | `onprem`, `reserved`, `ondemand`, `spot` |

Detailed information regarding `interconnect-topology` can be documented separately, but the value should remain implementation-agnostic in the `ResourceFlavor` name and labels.

CPU topology is intentionally not part of the default `ResourceFlavor` name. Exact NUMA shape, CPU model, core count, and thread count can split otherwise compatible accelerator pools into many narrower flavors and reduce flavor fungibility. CPU and memory capacity should normally be handled through Kubernetes resource requests, while CPU architecture remains part of the flavor name because it affects binary compatibility.

If CPU topology is required for a specific workload, it can be exposed through a separate optional label outside the default flavor identity.

## Example ResourceFlavor names

```text
nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-onprem
nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-reserved
nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-ondemand
nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-spot
nvidia-a100-80gb-pcie-2x2-eth-100g-amd64-onprem
```

## ResourceFlavor examples

This strategy is label-centric.

`ResourceFlavor` names provide a clear operational identity, but workloads should select hardware through Kubernetes scheduling fields. In practice, hardware selection should be expressed using only:

- `.nodeSelector`
- `.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution`

Only `spot` capacity is tainted. Non-spot availability tiers such as `onprem`, `reserved`, and `ondemand` are represented only as labels.

This keeps the model simple:

- hardware selection is expressed through labels
- common availability tiers do not require tolerations
- `spot` requires explicit opt-in because workloads may be interrupted

### Example ResourceFlavors

```yaml
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-onprem
spec:
  nodeLabels:
    example.com/accelerator-vendor: nvidia
    example.com/accelerator-name: h100-sxm
    example.com/accelerator-interconnect-topology: 2x4
    example.com/network-fabric: ib
    example.com/network-fabric-generation: ndr
    example.com/network-bandwidth: 400g
    kubernetes.io/arch: amd64
    example.com/availability: onprem
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-reserved
spec:
  nodeLabels:
    example.com/accelerator-vendor: nvidia
    example.com/accelerator-name: h100-sxm
    example.com/accelerator-interconnect-topology: 2x4
    example.com/network-fabric: ib
    example.com/network-fabric-generation: ndr
    example.com/network-bandwidth: 400g
    kubernetes.io/arch: amd64
    example.com/availability: reserved
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-ondemand
spec:
  nodeLabels:
    example.com/accelerator-vendor: nvidia
    example.com/accelerator-name: h100-sxm
    example.com/accelerator-interconnect-topology: 2x4
    example.com/network-fabric: ib
    example.com/network-fabric-generation: ndr
    example.com/network-bandwidth: 400g
    kubernetes.io/arch: amd64
    example.com/availability: ondemand
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-spot
spec:
  nodeLabels:
    example.com/accelerator-vendor: nvidia
    example.com/accelerator-name: h100-sxm
    example.com/accelerator-interconnect-topology: 2x4
    example.com/network-fabric: ib
    example.com/network-fabric-generation: ndr
    example.com/network-bandwidth: 400g
    kubernetes.io/arch: amd64
    example.com/availability: spot
  nodeTaints:
  - key: example.com/availability
    value: spot
    effect: NoSchedule
```

`example.com` is a placeholder DNS subdomain.

Replace it with a DNS subdomain owned by your organization.

The corresponding nodes should carry the same labels as the `ResourceFlavor`. This keeps the relationship between the flavor and the underlying node pool explicit and easy to audit.

For example, an H100 on-prem node should have:

```bash
kubectl label nodes <node> \
  example.com/accelerator-vendor=nvidia \
  example.com/accelerator-name=h100-sxm \
  example.com/accelerator-interconnect-topology=2x4 \
  example.com/network-fabric=ib \
  example.com/network-fabric-generation=ndr \
  example.com/network-bandwidth=400g \
  example.com/availability=onprem
```

An H100 spot node should carry the same hardware labels and the same spot taint:

```bash
kubectl label nodes <node> \
  example.com/accelerator-vendor=nvidia \
  example.com/accelerator-name=h100-sxm \
  example.com/accelerator-interconnect-topology=2x4 \
  example.com/network-fabric=ib \
  example.com/network-fabric-generation=ndr \
  example.com/network-bandwidth=400g \
  example.com/availability=spot

kubectl taint nodes <node> \
  example.com/availability=spot:NoSchedule
```

## ClusterQueue flavor ordering

The `flavors` list defines fallback preference. Kueue evaluates the list from top to bottom and admits a workload to the first compatible flavor with available quota.

This makes flavor ordering an important scheduling policy. If a workload can match multiple flavors, the ordering decides which type of capacity is consumed first.

### Ordering strategy: Availability first

Use Availability first ordering when you want to consume the most controlled and already-committed capacity before falling back to more elastic or interruptible capacity.

The recommended order is:

```text
onprem -> reserved -> ondemand -> spot
```

This order makes the capacity policy explicit:

- `onprem` is consumed first because it is usually the most controlled and already committed capacity.
- `reserved` is next because it is still committed capacity, but may come from a different pool or provider contract.
- `ondemand` is used after committed capacity because it is more elastic and may have higher marginal cost.
- `spot` is last because it is interruptible and should require explicit opt-in from workloads.

Within each availability tier, accelerators should be ordered from less capable or older to more capable or newer. This allows flexible workloads to consume more widely available hardware first, while still keeping premium accelerators available for workloads that explicitly need them.

```yaml
resourceGroups:
- coveredResources:
  - nvidia.com/gpu
  flavors:
  # onprem
  - name: nvidia-a100-80gb-pcie-2x2-eth-100g-amd64-onprem
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 32

  - name: nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-onprem
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 64

  # reserved
  - name: nvidia-a100-80gb-pcie-2x2-eth-100g-amd64-reserved
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 32

  - name: nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-reserved
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 64

  # ondemand
  - name: nvidia-a100-80gb-pcie-2x2-eth-100g-amd64-ondemand
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 32

  - name: nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-ondemand
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 32

  # spot
  - name: nvidia-a100-80gb-pcie-2x2-eth-100g-amd64-spot
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 32

  - name: nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-spot
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 128
```

Add a CPU `ResourceFlavor` to the same `ClusterQueue` so CPU-only workloads do not consume CPU and memory from GPU nodes. With proper `ResourceFlavor` `nodeLabels`, CPU-only workloads select CPU nodes, while GPU workloads select GPU flavors.

## User scheduling examples

Users should express only the hardware requirements that actually matter for the workload. The less specific the selector is, the more flexibility Kueue has to place the workload on an available compatible flavor.

### Select by accelerator name

If the user wants any H100 SXM node regardless of availability tier, interconnect topology, or network fabric, they should select by `accelerator-name`:

```yaml
nodeSelector:
  example.com/accelerator-name: h100-sxm
```

In multi-vendor environments, users can also include `accelerator-vendor` when they want to be explicit:

```yaml
nodeSelector:
  example.com/accelerator-vendor: nvidia
  example.com/accelerator-name: h100-sxm
```

Kueue may then admit the workload to any matching H100 `ResourceFlavor` in the `ClusterQueue`, such as `onprem`, `reserved`, or `ondemand`, depending on quota, flavor ordering, and `ClusterQueue` policy.

### Select by accelerator name and availability

If the user requires H100 SXM on-prem only:

```yaml
nodeSelector:
  example.com/accelerator-vendor: nvidia
  example.com/accelerator-name: h100-sxm
  example.com/availability: onprem
```

This workload can only be admitted to `ResourceFlavor`s that match these labels.

### Select an exact accelerator bundle

If a workload requires a precise combination of accelerator vendor, accelerator model, interconnect topology, network fabric, and CPU architecture, it can select all of those labels explicitly:

```yaml
nodeSelector:
  example.com/accelerator-vendor: nvidia
  example.com/accelerator-name: h100-sxm
  example.com/accelerator-interconnect-topology: 2x4
  example.com/network-fabric: ib
  example.com/network-fabric-generation: ndr
  example.com/network-bandwidth: 400g
  kubernetes.io/arch: amd64
```

This is useful when performance or compatibility depends on a specific hardware bundle, such as a particular accelerator interconnect structure or network fabric.

### Multiple accelerator types

If a workload can run on either H100 SXM or A100 PCIe, use `.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution`:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: example.com/accelerator-name
          operator: In
          values:
          - h100-sxm
          - a100-80gb-pcie
```

This gives Kueue more placement flexibility than a single exact `nodeSelector`, while still restricting placement to acceptable accelerator types.

### Multiple interconnect topologies

If a workload can run on more than one interconnect topology, use `.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution`:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: example.com/accelerator-interconnect-topology
          operator: In
          values:
          - 2x4
          - 4x2
          - 1x8
```

This allows the workload to remain flexible while still restricting placement to acceptable accelerator communication structures.


### Spot opt-in

If a workload can also run on spot capacity, it must explicitly tolerate the spot taint:

```yaml
nodeSelector:
  example.com/accelerator-name: h100-sxm

tolerations:
- key: example.com/availability
  operator: Equal
  value: spot
  effect: NoSchedule
```

Without this toleration, the workload can run on `onprem`, `reserved`, or `ondemand`, but not on `spot`.

This makes interruption risk explicit. Workloads only land on spot capacity when they intentionally opt in.

### No hardware preference

If a workload has no hardware preference and just needs a GPU as soon as possible, it can omit hardware selectors entirely.

In that case, the `ClusterQueue` flavor ordering becomes the effective placement policy.

This pattern is useful for:

- opportunistic or low-priority batch jobs
- development and testing workloads
- workloads where time-to-schedule matters more than hardware choice

## Summary

This `ResourceFlavor` strategy separates three concerns:

- `ResourceFlavor` names provide a readable operational identity for accelerator pools.
- Node labels define the actual hardware and availability properties used for scheduling.
- `ClusterQueue` flavor ordering defines fallback preference when multiple compatible flavors are available.

For heterogeneous accelerator clusters, this gives platform teams a consistent way to represent hardware diversity while giving users a simple contract: specify the hardware constraints that matter, and let Kueue choose the best available compatible flavor according to cluster policy.

## Future work

When Kueue's Dynamic Resource Allocation (DRA) integration becomes stable enough for production use, this document should be revisited.

DRA may provide a more expressive model for device-level requirements and allocation than encoding all accelerator properties through `ResourceFlavor` names and node labels. Future revisions should clarify how `ResourceFlavor`s, node labels, DRA `DeviceClass`es, and `ResourceClaim`s should coexist.
