## Background

Kubernetes [`ResourceQuota`](https://kubernetes.io/docs/concepts/policy/resource-quotas/) limits aggregate resource consumption within a namespace. If creating or updating an object would exceed a hard quota, the API server rejects the request.

Overprovisioning namespace quotas does not provide controlled capacity sharing. Kubernetes documents that when [cluster capacity is lower than the sum of namespace quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/#how-kubernetes-resourcequotas-work), resource contention is handled on a first-come-first-served basis. It also states that [`ResourceQuota` is independent of cluster capacity](https://kubernetes.io/docs/concepts/policy/resource-quotas/#quota-and-cluster-capacity), and that more complex policies can be implemented by a controller that watches quota usage and adjusts each namespace's hard quota limits.

Kueue adds a Workload admission and quota-management layer. [`ClusterQueue`s in the same `Cohort`](https://kueue.sigs.k8s.io/v0.18/docs/concepts/cluster_queue/#flavors-and-borrowing-semantics) can share unused quota. Kueue calls this borrowing when a Workload does not fit within its ClusterQueue's unused `nominalQuota`, but fits within the unused quota in the Cohort and the ClusterQueue's `borrowingLimit`. A [`Cohort` can also define additional `nominalQuota`](https://kueue.sigs.k8s.io/v0.18/docs/concepts/cohort/#configuring-quotas) as a shared pool for its ClusterQueues.

**Accelerator Capacity Orchestration Value (ACOV)** is the public-cloud reference-price equivalent of accelerator quota usage reported for admitted Workloads above a ClusterQueue's `nominalQuota` through Cohort quota sharing.

> **In simple terms:** ACOV uses public-cloud reference prices to estimate the value of accelerator quota admitted above each ClusterQueue's `nominalQuota`. Under a stable Cohort configuration, this above-nominal usage represents quota consumed through Cohort sharing.
>
> For example, [KEDA](https://keda.sh/docs/2.20/concepts/scaling-deployments/) can use a [Prometheus trigger](https://keda.sh/docs/2.20/scalers/prometheus/) based on request rate to reduce the replica groups of a Kueue-managed [LeaderWorkerSet (LWS)](https://kueue.sigs.k8s.io/docs/tasks/run/leaderworkerset/) when traffic falls. If the scale-down lowers the serving ClusterQueue's admitted usage and leaves quota available for lending, Kueue can admit a Workload in another ClusterQueue above its own `nominalQuota`. ACOV values only that above-nominal admitted quota for the time it remains admitted—not all quota released by the scale-down and not realized cost savings.

ACOV measures the quota-sharing effect rather than physical device activity. Overall accelerator utilization may remain unchanged even when Cohort sharing allows a ClusterQueue to use otherwise-unused capacity available in its Cohort.

## KPI calculation

Let $C$ be the measured Cohort scope and let $i$ identify an accelerator resource and `ResourceFlavor` pair.

For ClusterQueue $q$, define admitted quota above `nominalQuota` as:

```math
E_{q,i}(t)
=
\left[U^{\mathrm{adm}}_{q,i}(t)-N_{q,i}(t)\right]_+
```

where $[x]_+=\max(x,0)$.

The shared admitted quota-hours for pair $i$ are:

```math
H_i
=
\int_{t_0}^{t_1}
\sum_{q \in \mathcal{Q}_{C,i}(t)}
E_{q,i}(t)\,\mathrm{d}t_h
```

ACOV for the measured Cohort scope is:

```math
\mathrm{ACOV}_{C,[t_0,t_1]}
=
\sum_{i\in\mathcal{A}} H_iP_i
```

where:

- $\mathcal{A}$ is the set of accelerator resource and `ResourceFlavor` pairs included in the KPI.
- $\mathcal{Q}_{C,i}(t)$ is the set of ClusterQueues in scope $C$ that define pair $i$ at time $t$. For a hierarchical Cohort, the scope includes its descendant ClusterQueues.
- $U^{\mathrm{adm}}_{q,i}(t)$ is ClusterQueue $q$'s accelerator quota represented by admitted Workloads.
- $N_{q,i}(t)$ is its `nominalQuota`.
- $\mathrm{d}t_h$ denotes elapsed time measured in hours.
- $H_i$ is the shared admitted accelerator quota-hours for pair $i$.
- $P_i$ is the public-cloud reference price per unit-hour of accelerator resource $i$.

The positive part must be applied per ClusterQueue before summing. Moving it outside the ClusterQueue sum would offset usage above one ClusterQueue's `nominalQuota` against unused quota in another and could reduce a real sharing event to zero. Resource and `ResourceFlavor` pairs must also remain separate until each $H_i$ is multiplied by its own $P_i$.

The required values are exposed by the [optional Kueue metrics](https://kueue.sigs.k8s.io/v0.18/docs/reference/metrics/#optional-metrics):

- `kueue_cluster_queue_resource_usage`
- `kueue_cluster_queue_nominal_quota`

These metrics include `cohort`, `cluster_queue`, `flavor`, `resource`, and `replica_role` labels and require `metrics.enableClusterQueueResources: true`. Kueue's [usage implementation](https://github.com/kubernetes-sigs/kueue/blob/v0.18.2/pkg/cache/scheduler/cache.go#L3682-L3719) distinguishes `AdmittedResources` from `ReservedResources`; this KPI intentionally uses the admitted value exposed as `resource_usage`. It must not be interpreted as accelerator telemetry or proof that a device was actively computing.

If the intended KPI is instead quota held by all Workloads with a quota reservation, replace $U^{\mathrm{adm}}$ with reserved quota and use `kueue_cluster_queue_resource_reservation`. The admitted and reservation-based measurements must not be added together.

For attribution to Cohort sharing, calculate ACOV over intervals with stable `nominalQuota` and Cohort membership. Exclude or flag transition periods after quota reduction or Cohort movement until the ClusterQueue returns to a valid quota state. Kueue's [resource accounting code](https://github.com/kubernetes-sigs/kueue/blob/v0.18.2/pkg/cache/scheduler/resource_node.go#L904-L948) explicitly notes that such changes can produce over-admission. Use a fixed, versioned reference-price catalog for each interval as well.

When using hierarchical Cohorts, select one non-overlapping reporting scope so that the same descendant ClusterQueue is not counted at both a parent and child scope. In an HA deployment, use the authoritative `leader` series, or `standalone` in a non-HA deployment, rather than summing all `replica_role` values.

## AWS example

This example uses an NVIDIA A100 40 GB GPU as one accelerator type. ACOV itself applies to any accelerator represented by a Kueue resource and `ResourceFlavor`. A100 40 GB and A100 80 GB should use separate `ResourceFlavor`s and reference prices.

### 1. Estimate the accelerator reference price

Select a representative cloud instance for each accelerator model and use a consistent provider, region, operating system, and pricing model. AWS EC2 Linux On-Demand pricing in US East (N. Virginia) is used only as a consistent reference dataset for this example. On-Demand pricing provides a baseline without commitment terms or variable Spot pricing.

Because an accelerator instance also includes CPU, memory, and sometimes local storage, estimate the accelerator-only residual before dividing by the number of devices:

```math
P_i
=
\frac{
I_i
-(V_i \times P_{\mathrm{cpu}})
-(M_i \times P_{\mathrm{memory}})
-(S_i \times P_{\mathrm{storage}})
}{A_i}
```

where:

- $I_i$ is the instance's On-Demand hourly price.
- $V_i$ is its vCPU count.
- $M_i$ is its memory quantity.
- $S_i$ is its bundled local instance-storage quantity.
- $A_i$ is its accelerator count.

Set $S_i=0$ when the instance has no local instance storage. The quantities and coefficients must use consistent units. The [OpenCost AWS configuration](https://github.com/opencost/opencost/blob/develop/configs/aws.json) provides the following default coefficients for allocating the AWS node price among its resources:

| Component | Coefficient |
| --- | ---: |
| vCPU | 0.031611 |
| Memory | 0.004237 |
| Storage | 0.00005479452 |

The storage subtraction is optional and should be treated as a policy approximation. [EC2 instance store is included in the instance usage cost](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html), but subtracting an estimated storage portion avoids assigning all bundled local-storage value to the accelerator. The same unit convention must be used for every accelerator model.

AWS [`p4d.24xlarge`](https://aws.amazon.com/ec2/instance-types/p4/) provides eight NVIDIA A100 40 GB GPUs, 96 vCPUs, 1,152 GiB of memory, and 8,000 GB of local NVMe storage. On July 24, 2026, the [AWS EC2 On-Demand pricing data](https://b0.p.awsstatic.com/pricing/2.0/meteredUnitMaps/ec2/USD/current/ec2-ondemand-without-sec-sel/US%20East%20%28N.%20Virginia%29/Linux/index.json) listed its Linux price as **21.957642 USD per hour** in US East (N. Virginia).

Therefore:

```math
\begin{aligned}
P_{\mathrm{A100\text{-}40GB}}
&=
\frac{
21.957642
-(96 \times 0.031611)
-(1152 \times 0.004237)
-(8000 \times 0.00005479452)
}{8}\\
&=
\frac{
21.957642-3.034656-4.881024-0.43835616
}{8}\\
&=
1.70045073\ \mathrm{USD/A100\text{-}40GB\text{-}hour}
\end{aligned}
```

The resulting A100 40 GB reference price is approximately **1.70 USD per accelerator-hour**. It is a cost-aware policy approximation, not a separately billed AWS accelerator price.

### 2. Calculate ACOV

Assume the A100 40 GB `ResourceFlavor` remains in the following state for five hours:

| ClusterQueue | Admitted usage | `nominalQuota` | Usage above nominal |
| --- | ---: | ---: | ---: |
| A | 16 | 8 | 8 |
| B | 2 | 10 | 0 |

The shared admitted accelerator quota-hours are:

```math
H_{\mathrm{A100\text{-}40GB}}
=\left([16-8]_+ + [2-10]_+\right)\times5
=40\ \mathrm{A100\text{-}40GB\ quota\text{-}hours}
```

Applying the reference price:

```math
\mathrm{ACOV}
=40\times1.70045073
=68.0180292\ \mathrm{USD}
```

ACOV is therefore approximately **68.02 USD** for this period. For multiple accelerator models, calculate the value for each accelerator resource and `ResourceFlavor` pair and sum the results.

ACOV is a reference-price valuation of accelerator quota usage reported for admitted Workloads above ClusterQueue `nominalQuota`. It is not actual cloud spend, on-premises infrastructure cost, realized cost savings, workload runtime, or accelerator utilization.

## Complementary workload metrics

ACOV should be evaluated alongside workload-level performance metrics. For serving, quota sharing should count as successful only while service SLOs remain satisfied, using metrics such as [time to first token, inter-token latency, and token throughput](https://docs.vllm.ai/en/stable/features/per_request_metrics/). For training, the primary validation metric should be [time to target quality](https://mlcommons.org/benchmarks/training/), and quota sharing should count as successful only when it preserves or reduces the wall-clock time required to reach an agreed workload-specific quality target. ACOV should therefore not be treated as a standalone production success metric.

## Future work

### Dynamic quota orchestration

Kueue is discussing [Dynamic Quota Orchestration](https://github.com/kubernetes-sigs/kueue/issues/12382), which would adjust quota in response to available capacity. The proposal is still open. A [preliminary maintainer design sketch](https://github.com/kubernetes-sigs/kueue/issues/12382#issuecomment-4782274612) allows dynamically managed quota to target either a ClusterQueue or a Cohort, so its effects could overlap with Cohort sharing.

The current formula therefore covers only admitted quota above ClusterQueue `nominalQuota` under the Cohort-sharing semantics described above. Dynamic quota effects should remain future work until Kueue exposes stable semantics and metrics that allow them to be attributed without double-counting quota-hours already included in ACOV.
