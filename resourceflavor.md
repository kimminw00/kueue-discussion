# ResourceFlavor strategy for heterogeneous accelerator clusters

This document proposes a consistent strategy for naming and using Kueue `ResourceFlavor` resources in heterogeneous accelerator clusters.

The goal is to make accelerated compute resources easier to manage across different hardware classes, topology types, network fabrics, CPU architectures, and availability tiers. With a consistent naming convention and label model, platform teams can make scheduling behavior more predictable while giving users a simple way to express hardware requirements.

## ResourceFlavor naming convention

Use the following naming pattern for accelerator-backed `ResourceFlavor`s:

```text
<accelerator-vendor>-<accelerator-name>-<accelerator-topology>-<network-fabric>-<cpu-arch>-<availability>
```

This name is intentionally descriptive. It encodes the major scheduling-relevant properties of the node pool directly into the `ResourceFlavor` name, while the actual scheduling match is still driven by node labels.

### ResourceFlavor naming table

| Component              | Examples                                               |
| ---------------------- | ------------------------------------------------------ |
| `accelerator-vendor`   | `nvidia`, `amd`, `google`, `aws`, `intel`              |
| `accelerator-name`     | `h100-sxm`, `a100-80gb-pcie`, `h200-nvl`               |
| `accelerator-topology` | `2x4-nvswitch`, `1x4-nvlink-mesh`, `2x2-nvlink-bridge` |
| `network-fabric`       | `ib`, `eth`                                            |
| `cpu-arch`             | `amd64`, `arm64`                                       |
| `availability`         | `onprem`, `reserved`, `ondemand`, `spot`               |

Detailed information regarding `accelerator-topology` can be found in this [article](https://frankdenneman.ai/2026-03-27-Understanding-Multi-GPU-Topologies-Within-a-Single-Host).

### Example ResourceFlavor names

```text
nvidia-h100-sxm-2x4-nvswitch-ib-amd64-onprem
nvidia-h100-sxm-2x4-nvswitch-ib-amd64-reserved
nvidia-h100-sxm-2x4-nvswitch-ib-amd64-ondemand
nvidia-h100-sxm-2x4-nvswitch-ib-amd64-spot
nvidia-a100-80gb-pcie-2x2-nvlink-bridge-eth-amd64-onprem
```

## ResourceFlavor examples

This strategy is label-centric.

`ResourceFlavor` names provide a clear operational identity, but workloads should select hardware through Kubernetes scheduling fields. In practice, hardware selection should be expressed using only:

* `.nodeSelector`
* `.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution`

Only `spot` capacity is tainted. Non-spot availability tiers such as `onprem`, `reserved`, and `ondemand` are represented only as labels.

This keeps the model simple:

* hardware selection is expressed through labels
* common availability tiers do not require tolerations
* `spot` requires explicit opt-in because workloads may be interrupted

### Example ResourceFlavors

```yaml
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: nvidia-h100-sxm-2x4-nvswitch-ib-amd64-onprem
spec:
  nodeLabels:
    example.com/accelerator-vendor: nvidia
    example.com/accelerator-name: h100-sxm
    example.com/accelerator-topology: 2x4-nvswitch
    example.com/network-fabric: ib
    kubernetes.io/arch: amd64
    example.com/availability: onprem
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: nvidia-h100-sxm-2x4-nvswitch-ib-amd64-reserved
spec:
  nodeLabels:
    example.com/accelerator-vendor: nvidia
    example.com/accelerator-name: h100-sxm
    example.com/accelerator-topology: 2x4-nvswitch
    example.com/network-fabric: ib
    kubernetes.io/arch: amd64
    example.com/availability: reserved
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: nvidia-h100-sxm-2x4-nvswitch-ib-amd64-ondemand
spec:
  nodeLabels:
    example.com/accelerator-vendor: nvidia
    example.com/accelerator-name: h100-sxm
    example.com/accelerator-topology: 2x4-nvswitch
    example.com/network-fabric: ib
    kubernetes.io/arch: amd64
    example.com/availability: ondemand
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: nvidia-h100-sxm-2x4-nvswitch-ib-amd64-spot
spec:
  nodeLabels:
    example.com/accelerator-vendor: nvidia
    example.com/accelerator-name: h100-sxm
    example.com/accelerator-topology: 2x4-nvswitch
    example.com/network-fabric: ib
    kubernetes.io/arch: amd64
    example.com/availability: spot
  nodeTaints:
  - key: example.com/availability
    value: spot
    effect: NoSchedule
```

> `example.com` is a placeholder DNS subdomain.
> Replace it with a DNS subdomain owned by your organization.

The corresponding nodes should carry the same labels as the `ResourceFlavor`. This keeps the relationship between the flavor and the underlying node pool explicit and easy to audit.

For example, an H100 on-prem node should have:

```bash
kubectl label nodes <node> \
  example.com/accelerator-vendor=nvidia \
  example.com/accelerator-name=h100-sxm \
  example.com/accelerator-topology=2x4-nvswitch \
  example.com/network-fabric=ib \
  example.com/availability=onprem
```

An H100 spot node should carry the same hardware labels and the same spot taint:

```bash
kubectl label nodes <node> \
  example.com/accelerator-vendor=nvidia \
  example.com/accelerator-name=h100-sxm \
  example.com/accelerator-topology=2x4-nvswitch \
  example.com/network-fabric=ib \
  example.com/availability=spot

kubectl taint nodes <node> \
  example.com/availability=spot:NoSchedule
```

## ClusterQueue flavor ordering

The `flavors` list defines fallback preference. Kueue evaluates the list from top to bottom and admits a workload to the first compatible flavor with available quota.

This makes flavor ordering an important scheduling policy. If a workload can match multiple flavors, the ordering decides which type of capacity is consumed first.

### Ordering strategy: Availability first

Use **Availability first** ordering when you want to consume the most controlled and already-committed capacity before falling back to more elastic or interruptible capacity.

The recommended order is:

```text
onprem -> reserved -> ondemand -> spot
```

This order makes the capacity policy explicit:

* `onprem` is consumed first because it is usually the most controlled and already committed capacity.
* `reserved` is next because it is still committed capacity, but may come from a different pool or provider contract.
* `ondemand` is used after committed capacity because it is more elastic and may have higher marginal cost.
* `spot` is last because it is interruptible and should require explicit opt-in from workloads.

Within each availability tier, accelerators should be ordered from less capable or older to more capable or newer. This allows flexible workloads to consume more widely available hardware first, while still keeping premium accelerators available for workloads that explicitly need them.

```yaml
resourceGroups:
- coveredResources:
  - nvidia.com/gpu
  flavors:
  # onprem
  - name: nvidia-a100-80gb-pcie-2x2-nvlink-bridge-eth-amd64-onprem
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 32

  - name: nvidia-h100-sxm-2x4-nvswitch-ib-amd64-onprem
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 64

  # reserved
  - name: nvidia-a100-80gb-pcie-2x2-nvlink-bridge-eth-amd64-reserved
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 32

  - name: nvidia-h100-sxm-2x4-nvswitch-ib-amd64-reserved
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 64

  # ondemand
  - name: nvidia-a100-80gb-pcie-2x2-nvlink-bridge-eth-amd64-ondemand
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 32

  - name: nvidia-h100-sxm-2x4-nvswitch-ib-amd64-ondemand
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 32

  # spot
  - name: nvidia-a100-80gb-pcie-2x2-nvlink-bridge-eth-amd64-spot
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 32

  - name: nvidia-h100-sxm-2x4-nvswitch-ib-amd64-spot
    resources:
    - name: nvidia.com/gpu
      nominalQuota: 128
```

Add a CPU `ResourceFlavor` to the same `ClusterQueue` so CPU-only workloads do not consume CPU and memory from GPU nodes. With proper `ResourceFlavor` `nodeLabels`, CPU-only workloads select CPU nodes, while GPU workloads select GPU flavors.

## User scheduling examples

Users should express only the hardware requirements that actually matter for the workload. The less specific the selector is, the more flexibility Kueue has to place the workload on an available compatible flavor.

### Select by accelerator name

If the user wants any H100 SXM node regardless of availability tier, they should select by `accelerator-name`:

```yaml
nodeSelector:
  example.com/accelerator-name: h100-sxm
```

Kueue may then admit the workload to any matching H100 `ResourceFlavor` in the `ClusterQueue`, such as `onprem`, `reserved`, or `ondemand`, depending on quota, flavor ordering, and `ClusterQueue` policy.

### Select by accelerator name and availability

If the user requires H100 SXM on-prem only:

```yaml
nodeSelector:
  example.com/accelerator-name: h100-sxm
  example.com/availability: onprem
```

This workload can only be admitted to `ResourceFlavor`s that match both labels.

### Select an exact accelerator bundle

If a workload requires a precise combination of accelerator, topology, fabric, and CPU architecture, it can select all of those labels explicitly:

```yaml
nodeSelector:
  example.com/accelerator-name: h100-sxm
  example.com/accelerator-topology: 2x4-nvswitch
  example.com/network-fabric: ib
  kubernetes.io/arch: amd64
```

This is useful when performance or compatibility depends on a specific hardware bundle, such as a particular GPU topology or network fabric.

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

* opportunistic or low-priority batch jobs
* development and testing workloads
* workloads where time-to-schedule matters more than hardware choice

## Summary

This ResourceFlavor strategy separates three concerns:

* `ResourceFlavor` names provide a readable operational identity for accelerator pools.
* Node labels define the actual hardware and availability properties used for scheduling.
* `ClusterQueue` flavor ordering defines fallback preference when multiple compatible flavors are available.

For heterogeneous accelerator clusters, this gives platform teams a consistent way to represent hardware diversity while giving users a simple contract: specify the hardware constraints that matter, and let Kueue choose the best available compatible flavor according to cluster policy.
