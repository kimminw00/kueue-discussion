In multi-tenant accelerator clusters, a single project often needs two different scheduling contracts:

1. **Guaranteed capacity** for workloads that must run within secured project quota.
2. **Opportunistic capacity** for workloads that can use idle cluster resources but can tolerate preemption.

A practical way to model this is to create two `ClusterQueue`s per project:

```text
<project-name>-guaranteed
<project-name>-opportunistic
```

For example:

```text
project-a-guaranteed
project-a-opportunistic
project-b-guaranteed
project-b-opportunistic
```

This design is inspired by the `ClusterQueue` separation model discussed in [Batch Systems in Production with Kueue: Multi-Tenancy and Fungibility](https://www.youtube.com/watch?v=cEnor-oW9_s&t=1359s).

---

## Guaranteed ClusterQueue strategy

The **Guaranteed ClusterQueue** represents the project’s secured entitlement.

It is intended for workloads that should run predictably within the quota assigned to the project. In this strategy, guaranteed queues usually do **not borrow** from other queues by default. This makes the queue easier to reason about: if a workload is admitted through the guaranteed queue, it is running against the project’s own reserved capacity.

At the same time, unused guaranteed quota can still be shared with other queues through `lendingLimit`. This allows the cluster to stay highly utilized without weakening the project’s entitlement.

### Example

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: project-a-guaranteed
spec:
  cohortName: shared-compute-pool

  namespaceSelector:
    matchLabels:
      example.com/project: project-a
      kueue.x-k8s.io/managed-namespace: "true"

  queueingStrategy: BestEffortFIFO

  fairSharing:
    weight: "1"

  preemption:
    reclaimWithinCohort: Any
    withinClusterQueue: Never

  flavorFungibility:
    whenCanBorrow: TryNextFlavor
    whenCanPreempt: MayStopSearch

  resourceGroups:
  - coveredResources:
    - cpu
    - memory
    - pods
    - nvidia.com/gpu
    flavors:
    - name: nvidia-h100-sxm-2x4-nvswitch-ib-amd64-onprem
      resources:
      - name: cpu
        nominalQuota: "60"
        borrowingLimit: "0"
        lendingLimit: "30"
      - name: memory
        nominalQuota: "256Gi"
        borrowingLimit: "0"
        lendingLimit: "128Gi"
      - name: pods
        nominalQuota: "30"
        borrowingLimit: "0"
        lendingLimit: "0"
      - name: nvidia.com/gpu
        nominalQuota: "30"
        borrowingLimit: "0"
        lendingLimit: "15"
```

### Policy explanation

This policy expresses the following behavior:

* Guaranteed workloads run within secured project quota.
* `borrowingLimit: "0"` prevents the guaranteed queue from depending on borrowed resources.
* `lendingLimit` allows idle quota to be shared with other queues in the same cohort.
* `reclaimWithinCohort: Any` allows guaranteed workloads to reclaim quota that was temporarily borrowed by other queues.
* `withinClusterQueue: Never` prevents workloads inside the same guaranteed queue from preempting each other.
* `pods` quota acts as a concurrency and fragmentation guard. It is useful not only for capacity accounting, but also to discourage a pattern where users submit many tiny Pods to occupy scheduler slots or bypass the intended workload shape.

In short, this queue is optimized for **predictability first**, while still allowing unused capacity to contribute to overall cluster utilization.

### Suitable workloads

The guaranteed strategy is a good fit for:

* Production serving workloads.
* Training jobs that require secured accelerator quota.
* Time-sensitive workloads with predictable capacity requirements.
* Project-level quota guarantees.
* Workloads where unexpected preemption would be expensive or operationally risky.

---

## Opportunistic ClusterQueue strategy

The **Opportunistic ClusterQueue** represents idle-capacity usage.

It is intended for workloads that should run when the cluster has spare resources, but should not own guaranteed accelerator quota. Opportunistic workloads can improve utilization significantly, especially in heterogeneous accelerator environments where different projects may leave different accelerator types idle at different times.

Unlike the guaranteed queue, the opportunistic queue usually has `nominalQuota: "0"` for scarce accelerator resources. This makes the policy explicit: opportunistic workloads are admitted by using borrowed idle capacity, not by consuming a project entitlement.

### Example

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: project-a-opportunistic
spec:
  cohortName: shared-compute-pool

  namespaceSelector:
    matchLabels:
      example.com/project: project-a
      kueue.x-k8s.io/managed-namespace: "true"

  queueingStrategy: BestEffortFIFO

  fairSharing:
    weight: "1"

  admissionScope:
    admissionMode: UsageBasedAdmissionFairSharing

  preemption:
    reclaimWithinCohort: Never
    withinClusterQueue: Never

  flavorFungibility:
    whenCanBorrow: TryNextFlavor
    whenCanPreempt: TryNextFlavor

  resourceGroups:
  - coveredResources:
    - cpu
    - memory
    - pods
    - nvidia.com/gpu
    flavors:
    - name: nvidia-h100-sxm-2x4-nvswitch-ib-amd64-onprem
      resources:
      - name: cpu
        nominalQuota: "0"
        lendingLimit: "0"
      - name: memory
        nominalQuota: "0"
        lendingLimit: "0"
      - name: pods
        nominalQuota: "30"
        borrowingLimit: "0"
        lendingLimit: "0"
      - name: nvidia.com/gpu
        nominalQuota: "0"
        lendingLimit: "0"
```

### Policy explanation

This policy makes opportunistic workloads useful, but clearly preemptible:

* Opportunistic workloads can use idle accelerator capacity.
* `nominalQuota: "0"` makes it clear that this queue does not own guaranteed accelerator quota.
* `reclaimWithinCohort: Never` prevents opportunistic workloads from reclaiming resources from other queues.
* Guaranteed workloads can reclaim their quota from opportunistic workloads when needed.
* `withinClusterQueue: Never` avoids preemption between workloads submitted to the same opportunistic queue.
* `UsageBasedAdmissionFairSharing` helps distribute idle capacity fairly when multiple users or namespaces submit to the same opportunistic queue.
* `pods` quota is still useful even when accelerator quota is opportunistic. It limits the number of admitted Pods and helps prevent the cluster from being filled with many very small Pods that consume scheduling capacity inefficiently.

In short, this queue is optimized for **utilization first**, while preserving the right of guaranteed queues to reclaim their secured quota.

### Suitable workloads

The opportunistic strategy is a good fit for:

* Interruptible training jobs.
* Batch inference jobs.
* Development and experimentation workloads.
* Hyperparameter sweeps that can tolerate preemption.
* Low-priority research workloads.
* Workloads where higher cluster utilization is more important than strict completion-time guarantees.

It is usually not a good fit for production serving, deadline-sensitive jobs, or workloads with very high checkpoint/restart cost.

---

## Fair Sharing policy

Fair Sharing is most useful when queues compete for **borrowed capacity**.

In this design, guaranteed queues usually have:

```yaml
borrowingLimit: "0"
```

That means guaranteed queues do not normally participate in borrowed-capacity competition. They primarily consume their own nominal quota.

Opportunistic queues are different. They usually have:

```yaml
nominalQuota: "0"
```

Most of their usage is borrowed, so Fair Sharing matters much more for opportunistic capacity.

### Equal opportunistic policy

Use an equal opportunistic policy when all projects should have the same relative claim to idle borrowed capacity.

This is usually the best default. Opportunistic queues do not own guaranteed accelerator quota, so equal weights should be understood as an idle-capacity sharing policy, not as a guarantee. When multiple opportunistic queues are competing for the same unused capacity, equal weights tell Fair Sharing to treat them symmetrically.

```yaml
fairSharing:
  weight: "1"
```

Example policy:

```text
project-a-opportunistic weight: 1
project-b-opportunistic weight: 1
project-c-opportunistic weight: 1
```

Policy meaning:

```text
When idle accelerator capacity is contested, each opportunistic queue has the same relative claim to that borrowed capacity.
```

This does **not** mean that each project is guaranteed the same amount of accelerator capacity. Guaranteed capacity is still modeled by the guaranteed queues through `nominalQuota`. Equal opportunistic weights only mean that leftover capacity is shared fairly when opportunistic queues are borrowing from the shared pool.

### Weighted opportunistic policy

Use a weighted opportunistic policy when idle capacity should still be shared, but some projects or workload classes should have a stronger relative claim to borrowed idle capacity.

The important point is that an opportunistic queue's weight does **not** decide how much guaranteed quota the project receives. Guaranteed capacity should still be modeled in the guaranteed queue through `nominalQuota`.

Instead, the opportunistic weight is a policy value for contention among opportunistic queues. It decides which queue should get a stronger preference when multiple opportunistic queues are trying to borrow the same leftover capacity.

In other words:

```text
opportunistic weight is not about guaranteed quota.
opportunistic weight is about priority when borrowing idle capacity is contested.
```

Example policy:

```text
project-a-opportunistic weight: 2
project-b-opportunistic weight: 1
project-c-opportunistic weight: 1
```

Policy meaning:

```text
When idle capacity is contested, project-a has roughly twice the relative claim of project-b or project-c.
```

This does **not** mean that `project-a` is guaranteed twice as much accelerator capacity. It only means that, while opportunistic queues are borrowing unused capacity, Fair Sharing should bias the borrowed-capacity allocation toward `project-a` according to the configured weight.

Valid reasons may include:

* A project has an explicitly agreed higher share of opportunistic idle-capacity usage.
* A project pays for a larger portion of the shared accelerator pool.
* A project belongs to a higher compute service tier.

In general, start with equal weights and introduce weighted policy only when the operational policy is clear. A very low weight can be useful for background work, but it also increases the chance of starvation when the cluster is busy. Use `0` carefully because it makes the queue extremely weak whenever borrowed capacity is contested.

---

## Admission Fair Sharing policy

Admission Fair Sharing is useful when multiple `LocalQueue`s or users share a single opportunistic `ClusterQueue`.

For example:

```text
project-a-opportunistic
  <- user-a/opportunistic
  <- user-b/opportunistic
  <- user-c/opportunistic
```

In this case, users are competing for idle capacity. Historical usage should influence admission order so that one user does not continuously dominate the opportunistic pool.

This section focuses on how to choose the values in `admissionFairSharing`. These values define how historical usage is measured, how quickly old usage decays, and how strongly each resource contributes to the usage score.

A practical approach is to make the usage score cost-aware. OpenCost's AWS pricing configuration provides default allocation values for CPU, RAM, GPU, and storage. These values can be used as a starting point for `resourceWeights`, so Admission Fair Sharing accounts not only for how much resource was consumed, but also for the estimated cost of that usage.

A simple cost-oriented starting point is:

```yaml
admissionFairSharing:
  usageHalfLifeTime: "168h"
  usageSamplingInterval: "5m"
  resourceWeights:
    cpu: 0.031611
    memory: 0.004237
    nvidia.com/gpu: 0.95
```

The rationale for each value is:

* `usageHalfLifeTime: "168h"` uses a one-week half-life for historical usage. The policy goal is to balance recent usage and forgiveness. Seven days is long enough to discourage repeated opportunistic overuse within the same week, while still allowing older usage to gradually lose influence so users are not penalized indefinitely.
* `usageSamplingInterval: "5m"` samples usage every five minutes. The policy goal is to capture meaningful accelerator usage frequently enough for fair admission decisions, without making the usage score too noisy for short-lived workload changes.
* `cpu: 0.031611` uses the CPU allocation value from the same OpenCost AWS configuration.
* `memory: 0.004237` uses the RAM allocation value from the same OpenCost AWS configuration.
* `nvidia.com/gpu: 0.95` uses the default GPU allocation value from the OpenCost AWS pricing configuration. This is useful when all GPUs are treated as a single generic GPU resource.

Policy meaning:

```text
Admission Fair Sharing can be based on estimated resource cost.
GPU usage contributes the most because it has the highest cost weight.
CPU and memory usage still matter, but with lower cost-based weights.
Recent usage matters more than old usage because the usage score decays over time.
```

---

### Model-specific accelerator weights

For heterogeneous accelerator clusters, a single `nvidia.com/gpu` weight is often too coarse. A T4, L4, A10G, L40S, A100, H100, and H200 should not necessarily contribute the same amount to historical usage.

A better approach is to expose accelerator models as separate extended resources and assign each model its own `resourceWeight`.

Use an organization-owned extended resource namespace. In this example, `accelerator.example.com` is used as the platform-defined namespace:

```text
accelerator.example.com/nvidia-t4
accelerator.example.com/nvidia-l4
accelerator.example.com/nvidia-a10g
accelerator.example.com/nvidia-l40s
accelerator.example.com/nvidia-a100-40gb
accelerator.example.com/nvidia-h100-80gb
accelerator.example.com/nvidia-h200-141gb
```

In production, replace `example.com` with a DNS subdomain owned by your organization.

The domain prefix represents the platform-owned accelerator resource namespace:

```text
accelerator.example.com
```

The resource name after the slash describes the vendor and accelerator model:

```text
nvidia-h100-80gb
amd-mi300x
google-tpu-v5e
aws-trn2
```

This avoids using vendor-owned namespaces such as `nvidia.com/a100` or `amd.com/mi300x` for platform-defined accounting resources.

---

### Estimating model-specific weights

To estimate a model-specific accelerator weight, start from a representative instance type and subtract the non-accelerator portions first.

The most stable approximation subtracts CPU and memory from the instance hourly price:

```text
estimated_accelerator_cost_per_instance
= instance_hourly_price
  - (vcpu_count * cpu_weight)
  - (memory_gib * memory_weight)

estimated_accelerator_cost_per_device
= estimated_accelerator_cost_per_instance / accelerator_count
```

Using the OpenCost AWS defaults:

```text
cpu_weight    = 0.031611
memory_weight = 0.004237
```

For accelerator instances with bundled local NVMe instance store, an optional storage adjustment can also be applied:

```text
estimated_accelerator_cost_per_instance
= instance_hourly_price
  - (vcpu_count * cpu_weight)
  - (memory_gib * memory_weight)
  - (local_instance_store_gib * storage_weight)

estimated_accelerator_cost_per_device
= estimated_accelerator_cost_per_instance / accelerator_count
```

Using the OpenCost AWS storage value:

```text
storage_weight = 0.00005479452
```

This storage adjustment should be treated as a policy approximation. EC2 instance store is included in the instance usage cost and is not billed as a separate line item. However, subtracting an estimated storage portion can avoid assigning too much of a large accelerator instance's bundled local storage value to the accelerator weight.

Do not subtract network egress, NAT gateway traffic, EBS volume cost, or EFA from the EC2 instance hourly price.

OpenCost's AWS configuration includes network and NAT gateway values, but those represent usage-based network costs, not fixed compute resources bundled into the EC2 instance hourly price. EBS is also billed separately from the instance. EFA should not be subtracted either, because it is an optional EC2 networking feature available on supported instances at no additional cost.

---

### Example residual-cost calculation

For example, if a representative accelerator instance has:

```text
instance_hourly_price = 1.006
vcpu_count            = 4
memory_gib            = 16
local_instance_store  = 250
accelerator_count     = 1
```

Then:

```text
estimated_accelerator_cost
= 1.006
  - (4 * 0.031611)
  - (16 * 0.004237)
  - (250 * 0.00005479452)

= 1.006
  - 0.126444
  - 0.067792
  - 0.013699

= 0.798065
```

So the model-specific weight can be written as:

```yaml
resourceWeights:
  cpu: 0.031611
  memory: 0.004237
  accelerator.example.com/nvidia-a10g: 0.80
```

This value is not an exact billing price. It is a cost-aware policy weight derived from the representative instance price after subtracting estimated CPU, memory, and local storage portions.

---

### Example model-specific resource weights

Using the same residual-cost method, model-specific weights can be estimated from representative accelerator instances:

```yaml
admissionFairSharing:
  usageHalfLifeTime: "168h"
  usageSamplingInterval: "5m"
  resourceWeights:
    cpu: 0.031611
    memory: 0.004237
    accelerator.example.com/nvidia-k80: 0.52
    accelerator.example.com/nvidia-t4: 0.32
    accelerator.example.com/nvidia-l4: 0.60
    accelerator.example.com/nvidia-a10g: 0.80
    accelerator.example.com/nvidia-l40s: 1.59
    accelerator.example.com/nvidia-v100: 2.55
    accelerator.example.com/nvidia-a100-40gb: 1.70
    accelerator.example.com/nvidia-h100-80gb: 4.83
    accelerator.example.com/nvidia-h200-141gb: 5.86
```

---

## LocalQueue strategy

Users should submit workloads to `LocalQueue`s, not directly to `ClusterQueue`s.

Create two `LocalQueue`s per namespace:

```text
default       -> <project-name>-guaranteed
opportunistic -> <project-name>-opportunistic
```

Example:

```yaml
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata:
  namespace: project-a
  name: default
spec:
  clusterQueue: project-a-guaranteed
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata:
  namespace: project-a
  name: opportunistic
spec:
  clusterQueue: project-a-opportunistic
```

With LocalQueue defaulting, `default` can be used as the namespace default queue.

Users only need to specify a queue when they intentionally want opportunistic capacity:

```yaml
metadata:
  labels:
    kueue.x-k8s.io/queue-name: opportunistic
```

This gives users a simple and explicit contract:

```text
default       = secured project quota
opportunistic = idle capacity, may be preempted
```

---

## Recommended baseline

```text
ClusterQueues:
- <project>-guaranteed
- <project>-opportunistic

Guaranteed:
- nominalQuota > 0
- borrowingLimit: 0
- lendingLimit: set based on how much idle quota can be shared
- queueingStrategy: BestEffortFIFO for mixed training and serving
- fairSharing.weight: 1
- reclaimWithinCohort: Any
- withinClusterQueue: Never
- Admission Fair Sharing: off by default, enable only when needed
- pods quota: set as a guardrail against excessive tiny-Pod submissions

Opportunistic:
- nominalQuota: 0 for scarce accelerator resources
- borrowingLimit omitted or set to a positive cap depending on policy
- queueingStrategy: BestEffortFIFO
- fairSharing.weight: 1 if projects are equal
- fairSharing.weight: lower if this queue should be weaker
- reclaimWithinCohort: Never
- withinClusterQueue: Never
- Admission Fair Sharing: enabled
- pods quota: set to limit concurrency and reduce scheduler fragmentation

Fair Sharing:
- use for borrowed-capacity fairness
- start with equal weights
- use different weights only with explicit policy reasons

Admission Fair Sharing:
- use cost-aware resourceWeights
- start with generic nvidia.com/gpu only if all GPUs can be treated equally
- use model-specific extended resources when accelerator models have different cost, scarcity, or business value

LocalQueue:
- default -> guaranteed
- opportunistic -> opportunistic
```

---

## Summary

The two-queue model separates quota guarantees from idle-capacity utilization:

* **Guaranteed ClusterQueue** is for secured, predictable project capacity.
* **Opportunistic ClusterQueue** is for idle capacity that can be preempted.
* **Fair Sharing** balances borrowed capacity across projects.
* **Admission Fair Sharing** balances opportunistic usage across users or local queues.
* **Pod quota and pod resource weights** provide a practical guardrail against excessive small-Pod submissions and help keep scheduling behavior aligned with the intended workload policy.

This pattern works well for heterogeneous accelerator clusters because it lets each project keep a clear entitlement while still allowing idle GPUs, TPUs, or other accelerator flavors to be used efficiently.
