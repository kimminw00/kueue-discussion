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
    - name: nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-onprem
      resources:
      - name: cpu
        nominalQuota: "60"
        borrowingLimit: "0"
        lendingLimit: "0"
      - name: memory
        nominalQuota: "256Gi"
        borrowingLimit: "0"
        lendingLimit: "0"
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
* To keep scheduling simple, only idle GPU quota is shared through `lendingLimit`; CPU and memory are kept as supporting quota for the guaranteed queue.
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
    - name: nvidia-h100-sxm-2x4-ib-ndr-400g-amd64-onprem
      resources:
      - name: cpu
        nominalQuota: "120"
        borrowingLimit: "0"
        lendingLimit: "0"
      - name: memory
        nominalQuota: "512Gi"
        borrowingLimit: "0"
        lendingLimit: "0"
      - name: pods
        nominalQuota: "30"
        borrowingLimit: "0"
        lendingLimit: "0"
      - name: nvidia.com/gpu
        nominalQuota: "0"
        borrowingLimit: "16"
        lendingLimit: "0"
```

### Policy explanation

This policy makes opportunistic workloads useful, but clearly preemptible:

* Opportunistic workloads can use idle accelerator capacity.
* To keep scheduling simple, CPU and memory have their own positive `nominalQuota`, while only GPU is treated as borrowed opportunistic capacity.
* `nominalQuota: "0"` makes it clear that this queue does not own guaranteed accelerator quota.
* `borrowingLimit` caps how much idle cohort capacity this queue can consume.
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

## Preemption and borrowed capacity policy

This design uses **Classic Preemption** as the default instead of enabling Preemption-based Fair Sharing immediately.

The main reason is workload stability.

This cluster is expected to run a mix of workload types, including:

```text
- large-scale distributed training
- opportunistic batch workloads
- LLM serving workloads
- LWS-based serving workloads
```

In this environment, preemption can be expensive. Preempting a large training job can waste GPU-hours if the job has not checkpointed recently. Preempting an LLM serving workload can trigger model reloads, worker restarts, warmup, and readiness recovery.

Because of that, the default preemption policy should be easy for users to understand and should avoid unnecessary disruption.

### Why Classic Preemption is the default

Classic Preemption matches the guaranteed/opportunistic queue model naturally.

```text
guaranteed queue    = owns nominal quota
opportunistic queue = uses idle or borrowed capacity
```

With Classic Preemption, preemption happens mainly when:

```text
- a guaranteed queue needs to reclaim its nominal quota
- a higher-priority workload needs capacity according to the configured policy
```

This gives users a clearer contract:

```text
opportunistic workloads may be preempted when guaranteed quota must be reclaimed.
```

That is expected behavior for opportunistic usage.

For example:

```text
project-a-guaranteed owns 32 GPUs.
project-b-opportunistic is temporarily borrowing idle GPUs.
project-a-guaranteed submits a workload that needs its quota back.
project-b-opportunistic may be preempted so project-a-guaranteed can reclaim its quota.
```

This is easier to explain because the preemption is tied to quota ownership.

### Why not Preemption-based Fair Sharing by default?

Preemption-based Fair Sharing is useful when the cluster wants to actively rebalance borrowed capacity across `ClusterQueue`s.

However, it changes the meaning of preemption.

With Preemption-based Fair Sharing enabled, a workload may be preempted not only because a guaranteed queue needs to reclaim quota, but also because Kueue is trying to move the cohort closer to equal or weighted fair share.

For example:

```text
project-a-opportunistic is running a large training workload using many borrowed GPUs.
project-b-opportunistic later submits a workload.
Kueue sees that project-a has a high share.
Kueue may preempt project-a's workload to rebalance fair share.
```

This can be correct from a fairness perspective, but it can be disruptive from the user's perspective.

The key difference is:

```text
Classic Preemption:
- mainly quota-reclaim or priority driven
- easier to explain to users
- better aligned with guaranteed vs opportunistic queue contracts

Preemption-based Fair Sharing:
- can also preempt for fair-share rebalancing
- may interrupt large borrowed-capacity workloads even when no guaranteed quota is being reclaimed
- better suited when the cluster is ready to tolerate fair-share-driven preemption
```

For small, checkpointable, interruptible workloads, that trade-off may be acceptable.

For large training workloads or LLM serving workloads, it can be too disruptive as a default policy.

### Resource monopolization guardrails

Choosing Classic Preemption still requires explicit guardrails so that one `ClusterQueue` does not consume shared GPU capacity for too long or at too large a scale.

This design uses three simple controls:

```text
borrowingLimit:
- caps how much idle cohort capacity a ClusterQueue can borrow

maximumExecutionTimeSeconds:
- limits how long an opportunistic workload can run

Admission Fair Sharing:
- improves admission fairness without preempting running workloads
```

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

## Recommended baseline

```text
ClusterQueues:
- <project>-guaranteed
- <project>-opportunistic

Guaranteed:
- nominalQuota > 0 for scarce accelerator resources
- borrowingLimit: 0
- lendingLimit: set based on how much idle quota can be shared
- queueingStrategy: BestEffortFIFO for mixed training and serving
- reclaimWithinCohort: Any
- withinClusterQueue: Never
- Admission Fair Sharing: off by default, enable only when needed
- pods quota: set as a guardrail against excessive tiny-Pod submissions

Opportunistic:
- nominalQuota: 0 for scarce accelerator resources
- borrowingLimit: set a bounded cap
- queueingStrategy: BestEffortFIFO
- reclaimWithinCohort: Never
- withinClusterQueue: Never
- Admission Fair Sharing: enabled when multiple LocalQueues or users share the queue
- maximumExecutionTimeSeconds: required
- pods quota: set to limit concurrency and reduce scheduler fragmentation

Admission Fair Sharing:
- use cost-aware resourceWeights
- start with generic nvidia.com/gpu only if all GPUs can be treated equally
- use model-specific extended resources when accelerator models have different cost, scarcity, or business value
- use this for non-preemptive fairness between LocalQueues

LocalQueue:
- default -> guaranteed
- opportunistic -> opportunistic
```

## Summary

The two-queue model separates quota guarantees from idle-capacity utilization:

* **Guaranteed ClusterQueue** is for secured, predictable project capacity.
* **Opportunistic ClusterQueue** is for idle capacity that can be preempted when guaranteed quota is reclaimed.
* **Classic Preemption** provides a clearer and more predictable preemption contract.
* **Borrowing limits** prevent a single opportunistic queue from consuming too much shared capacity.
* **Admission Fair Sharing** balances opportunistic usage across users or LocalQueues without preempting running workloads.
* **Maximum execution time** prevents opportunistic workloads from holding GPUs indefinitely.
* **Pod quota** provides a practical guardrail against excessive small-Pod submissions and helps keep scheduling behavior aligned with the intended workload policy.

This pattern works well for heterogeneous accelerator clusters because it lets each project keep a clear entitlement while still allowing idle GPUs, TPUs, or other accelerator flavors to be used efficiently.

## Future work

Preemption-based Fair Sharing can be reconsidered later if Kueue supports stronger workload protection for fair-share preemption.

In particular, the feature request [Support guaranteed minimum runtime before fair sharing preemption](https://github.com/kubernetes-sigs/kueue/issues/9876) proposes allowing administrators to configure a minimum duration that a workload must be admitted before it can be preempted by fair sharing.

If that kind of protection becomes available, Preemption-based Fair Sharing could be used in a less disruptive way.

For example, the platform could express a policy like:

```text
A workload must run for at least N minutes before it becomes eligible for fair-share-driven preemption.
```

That would make fair-share rebalancing safer for large training workloads because newly admitted jobs would have time to make meaningful progress before becoming preemption candidates.
