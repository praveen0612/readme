# Karpenter — Architecture Deep-Dive

Adobe Ad Cloud Monorepo — Node Autoscaling Review

## Table of Contents

- [1. Deployment Scope](#1-deployment-scope)
- [2. Core Provisioning Loop](#2-core-provisioning-loop)
- [3. Major Config Components — NodePool / EC2NodeClass](#3-major-config-components--nodepool--ec2nodeclass)
- [4. IAM Dependencies](#4-iam-dependencies)
- [5. Spot Interruption Handling](#5-spot-interruption-handling)
- [6. Cilium Relationship](#6-cilium-relationship)
- [7. Cluster-Autoscaler Status](#7-cluster-autoscaler-status)
- [8. How Data/Metrics Capture Works](#8-how-datametrics-capture-works)

---

## 1. Deployment Scope

Karpenter (Helm chart `karpenter-1.3.2`, controller image `karpenter/controller:1.4.0`) runs on the **same 7 EKS clusters as Cilium**: `eks-mgmt`, `eks-usw2-prod`, `eks-usw2-dev`, `eks-us-east-1`, `eks-euw1-prod`, `eks-aws-lab`, `eks-dev`.

Deployed via the identical ArgoCD GitOps pattern used across this fleet: `argo/ArgoCD/<cluster>/kube-system/karpenter.yaml` → `cloud/kube-system/<cluster>/karpenter`, with `directory.recurse: true` and `syncPolicy.automated.selfHeal: true`.

API version confirmed: **`karpenter.sh/v1`** (NodePool) and **`karpenter.k8s.aws/v1`** (EC2NodeClass) — the current stable APIs, not the deprecated v1alpha5 `Provisioner`/`AWSNodeTemplate`.

## 2. Core Provisioning Loop

```
Pod created, unschedulable (no node satisfies its requests/tolerations)
        │
        ▼
Karpenter controller (watches API server) sees the pending pod
        │
        ▼
Evaluates all NodePools — picks one whose requirements (arch, instance
family, AZ, capacity-type) can satisfy the pod's requests/tolerations
        │
        ▼
Calls EC2 Fleet API (via controller's IAM role) → launches the EXACT
instance type needed (no pre-sized ASG — this is Karpenter's core
architectural difference from cluster-autoscaler, which only scales
pre-defined, fixed-shape node groups up/down)
        │
        ▼
New EC2 node boots with a custom AMI; userData runs:
  /etc/eks/bootstrap.sh <cluster> --kubelet-extra-args
    "--register-with-taints=karpenter.sh/unregistered:NoExecute"
        │
        ▼
Node joins cluster tainted "unregistered" → Karpenter removes the taint
once the node is confirmed Ready → pending pod gets scheduled
        │
        ▼
Ongoing: Karpenter's disruption controller continuously evaluates live
nodes for consolidation (empty nodes, or underutilized nodes that could
be bin-packed onto fewer/cheaper nodes) and drains + terminates them,
throttled by each NodePool's disruption budget
```

## 3. Major Config Components — NodePool / EC2NodeClass

Real example (`cloud/kube-system/eks-dev/karpenter/templates/nodepools/trino-arm64-nodepool.yaml`):

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
spec:
  template:
    spec:
      requirements:
        - {key: kubernetes.io/arch, operator: In, values: ["arm64"]}
        - {key: karpenter.sh/capacity-type, operator: In, values: ["on-demand"]}
        - {key: node.kubernetes.io/instance-type, operator: In, values: [r6g.large, r6g.xlarge, r6g.2xlarge, r6g.4xlarge]}
        - {key: topology.kubernetes.io/zone, operator: In, values: [us-east-1a, us-east-1d, us-east-1e]}
      nodeClassRef: {group: karpenter.k8s.aws, kind: EC2NodeClass, name: default-arm64}
      expireAfter: Never
      taints:
      - {key: node.kubernetes.io/arch, value: arm64, effect: NoSchedule}
      - {key: node.kubernetes.io/workload, value: trino, effect: NoSchedule}
  limits: {cpu: 512, memory: 1024Gi}
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 10m
    budgets:
    - {duration: 15m, nodes: "20%", reasons: [Empty], schedule: '*/5 * * * *'}
    - {duration: 15m, nodes: "1", reasons: [Drifted], schedule: '*/5 * * * *'}
```

**Field-by-field meaning:**

| Field | Purpose |
|---|---|
| `requirements` | Hard constraints Karpenter must satisfy when picking an instance — architecture, capacity type, instance family/size, AZ |
| `taints` | Isolates this pool so only pods explicitly tolerating `workload=trino` land here, protecting dedicated capacity from being stolen by unrelated workloads |
| `expireAfter: Never` | Disables forced node recycling purely by node age |
| `limits` | Hard cap on total pool size (512 vCPU / 1024Gi here) |
| `disruption.consolidationPolicy` | `WhenEmpty` = only remove fully-idle nodes; `WhenEmptyOrUnderutilized` (used elsewhere) also bin-packs underused nodes |
| `disruption.budgets` | Throttles churn — e.g. only 1 "drifted" node touched per 5-minute window, preventing thundering-herd node replacement |

Other real NodePools found on this fleet: `dedicated-nodepool` (r7i/m7i/m7i-flex families) uses `consolidationPolicy: WhenEmptyOrUnderutilized` but with a schedule that **blocks underutilized-node consolidation from 06:00–23:00 daily** — bin-packing is only allowed overnight, to avoid mid-day workload disruption. ML-specific pools (`ssc-ml-nodepool`, `ssc-ml-sim-nodepool`, `ssc-dl-dqrf-nodepool`) exist with their own per-cluster caps defined in `infrastructure/kubernetes/helm/values/<cluster>/karpenter/values.yaml`.

**EC2NodeClass** (`default-amd64` / `default-arm64`):
- `amiFamily: Custom` with a **pinned `amiSelectorTerms: [{id: "ami-..."}]`** — this org bakes its own AMI (Puppet-managed, per `puppet_role`/`puppet_environment` tags baked into userData), not Karpenter's default AL2023 auto-resolution.
- `subnetSelectorTerms` / `securityGroupSelectorTerms` match by AWS tag or explicit SG IDs.
- `role:` references the node IAM role directly — Karpenter v1 creates/manages the EC2 instance profile itself from this role; you don't provision the instance profile separately.

## 4. IAM Dependencies

Two distinct IAM identities — do not conflate them:

| Identity | Mechanism | Purpose |
|---|---|---|
| **Karpenter controller** | IRSA via ServiceAccount annotation `eks.amazonaws.com/role-arn: arn:aws:iam::290999691900:role/KarpenterControllerRole-<cluster>` | Needs `ec2:RunInstances` / `CreateFleet` / `TerminateInstances` / `DescribeInstanceTypes` |
| **Worker nodes** | EC2NodeClass `role: arn:aws:iam::290999691900:role/k8s-worker-<cluster>.k8s.tubemogul.info` | Standard EKS worker-node permissions (ECR pull, CNI, kubelet↔API) |

Neither role's Terraform definition was found under `infrastructure/kubernetes/terraform/` in this repo — both are provisioned outside this repo's GitOps tree (likely Puppet or a separate IAM-management repo).

## 5. Spot Interruption Handling

**Not configured anywhere in this fleet — and that's explained, not a gap.** The vendored chart's `interruptionQueue` value is left empty/unset in every single per-cluster `values.yaml`; no SQS queue or EventBridge rule for spot-interruption/rebalance-recommendation events exists in this repo.

The reason: **every NodePool across all 7 clusters and every workload type constrains `karpenter.sh/capacity-type` to `["on-demand"]` only** — zero spot instance usage found anywhere. Since there's no spot capacity in play, the interruption-handling pipeline genuinely isn't needed today.

> **Latent risk to flag:** this would become a real, active gap the moment anyone adds a NodePool with `capacity-type: spot` without also wiring up the interruption queue — spot nodes could then be reclaimed by AWS with only 2 minutes' notice and no graceful-drain mechanism in place.

## 6. Cilium Relationship

Every NodePool has a **commented-out (disabled)** `startupTaints` entry for `node.cilium.io/agent-not-ready` — this is the standard safety mechanism recommended when running Cilium alongside Karpenter, which prevents pods from being scheduled onto a brand-new node before Cilium's agent has finished initializing there.

> **Latent risk to flag:** with this taint disabled, there's a theoretical window right after a new node joins where a pod could be scheduled before Cilium's networking is actually ready on that node, causing early connection failures. It's present in code (someone was aware of the pattern) but turned off — worth understanding whether that was a deliberate call or leftover from initial rollout.

## 7. Cluster-Autoscaler Status

**Fully replaced, not coexisting.** Cluster-autoscaler manifests still physically exist in the repo on `eks-mgmt`, `eks-dev`, and `eks-aws-lab`, but all three have `replicas: 0` — confirmed disabled, left as inert leftover configuration rather than removed. On `eks-usw2-prod`, `eks-usw2-dev`, `eks-us-east-1`, `eks-euw1-prod`, no cluster-autoscaler directory exists at all. There is no active coexistence anywhere in this fleet.

## 8. How Data/Metrics Capture Works

```
karpenter-controller pod
  exposes :8080/metrics (METRICS_PORT env var)
        │
        ▼
ServiceMonitor (same pattern as every other component in this fleet)
  scrapes port "http-metrics" → /metrics
        │
        ▼
Prometheus stores: karpenter_cluster_state_synced,
  karpenter_nodeclaims_created_total, karpenter_nodeclaims_registered_time_seconds,
  karpenter_nodeclaims_ready, karpenter_provisioner_scheduling_queue_depth, etc.
        │
        ├──► PrometheusRule → 4 alert pairs (warning + critical each):
        │      KarpenterClusterStateDesynced   (karpenter_cluster_state_synced == 0)
        │      KarpenterPodPending             (unschedulable >15m/30m)
        │      KarpenterNoProvisioningActivity (queue depth up, no nodeclaims created)
        │      KarpenterNodeClaimNotReady      (registered but not Ready in time)
        │
        └──► Grafana dashboard (vendored with the Helm chart, not custom-built):
               CPU/Memory allocation, node/pod distribution by AZ & NodePool,
               spot %, nodepool usage vs. limit, interruption messages,
               node readiness, cloud-provider error rate
```

**One org-specific addition, separate from Karpenter's own metrics**: each node's boot `userData` self-registers with 6 NLB target groups (for nginx-ingress) via `aws elbv2 register-targets`, and pushes custom metrics — `karpenter_lb_target_group_registration_status`, `karpenter_lb_target_group_registration_failures_total` — to the same **Pushgateway** flagged as a collision-risk area in the earlier Thanos/S3 research. This is confirmed as its real-world use case here: tracking node-boot-time load-balancer registration status, backed by a dedicated `KarpenterNodeTargetGroupRegistrationFailure` alert.

---

*Compiled from a read-only review of the adCloud monorepo (`cloud/kube-system/`, `argo/ArgoCD/`, vendored Helm chart, per-cluster `values.yaml`) — September 2026.*
