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
- [9. Fresh Setup — Production-Ready Karpenter on a New Cluster](#9-fresh-setup--production-ready-karpenter-on-a-new-cluster)

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

## 9. Fresh Setup — Production-Ready Karpenter on a New Cluster

**Framing:** this section is a recommended, production-grade setup plan built from this org's own working reference config (Sections 3–8 above), plus standard Karpenter best practices for gaps this fleet currently leaves open (interruption handling, startup taints). It is not a transcription of an existing internal runbook — treat it as a plan to review with your team, test in dev/uat first, and adapt to your actual cluster's tags/CIDRs/account IDs before touching prod.

### Step 0 — Prerequisites

Before installing Karpenter, the cluster needs:
- An EKS cluster with an **OIDC provider** enabled (required for IRSA — Karpenter's controller authenticates to AWS via IRSA, not static keys).
- Subnets and security groups **tagged for discovery**, e.g. `karpenter.sh/discovery: <cluster-name>` on every subnet/SG Karpenter should be allowed to use. This is what `subnetSelectorTerms`/`securityGroupSelectorTerms` in EC2NodeClass match against — get this tag applied consistently at the Terraform/VPC level before writing any EC2NodeClass.
- A **stable, non-Karpenter-managed node group** (e.g. a small on-demand managed node group or Fargate profile) to run the Karpenter controller itself on — Karpenter can't safely scale the very node it's running on down to zero, so it needs a floor of guaranteed capacity outside its own management.
- Decide your **AMI strategy now**: this org bakes a custom AMI (`amiFamily: Custom`, pinned `amiSelectorTerms`) via Puppet, matching their existing worker-node bootstrap process. If you don't have an equivalent custom-AMI pipeline, using `amiFamily: AL2023` (Karpenter's own managed AMI resolution) is simpler and lower-maintenance for a new setup — pick one and be consistent; don't mix both patterns on the same cluster.

### Step 1 — IAM (two separate roles, don't conflate them)

**Controller role** (IRSA):
```yaml
# Trust policy: allow the karpenter ServiceAccount (via OIDC) to assume this role
# Permissions policy needs, at minimum:
ec2:CreateFleet, ec2:RunInstances, ec2:TerminateInstances,
ec2:DescribeInstances, ec2:DescribeInstanceTypes, ec2:DescribeImages,
ec2:DescribeSubnets, ec2:DescribeSecurityGroups, ec2:DescribeLaunchTemplates,
ec2:CreateTags, ec2:CreateLaunchTemplate, ec2:DeleteLaunchTemplate,
iam:PassRole (scoped to the node role only),
eks:DescribeCluster,
pricing:GetProducts (for spot/on-demand price lookups),
sqs:ReceiveMessage, sqs:DeleteMessage, sqs:GetQueueUrl, sqs:GetQueueAttributes
  (only if interruption handling is enabled — see Step 3)
```
Bind this to the `karpenter` ServiceAccount via `eks.amazonaws.com/role-arn`, matching the pattern already used across this org's 7 clusters.

**Node role**: a standard EKS worker-node IAM role (the three AWS-managed policies: `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly`, plus SSM if you want Session Manager access to nodes). Reference this role directly in every EC2NodeClass's `spec.role` field — Karpenter v1 creates the instance profile for you from this role, you do not need to pre-create an instance profile.

### Step 2 — Install Karpenter (Helm)

```bash
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "1.4.0" \
  --namespace kube-system \
  --set settings.clusterName=<cluster-name> \
  --set settings.interruptionQueue=<cluster-name>-karpenter \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<controller-role-arn> \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set replicas=2 \
  --wait
```
`replicas: 2` (not the default 1) is a production-readiness call — Karpenter supports leader-election HA, and running a single replica means any pod eviction/node issue on that one pod briefly halts all node provisioning cluster-wide. This org's own clusters should be checked for whether they already do this; if not, it's worth adopting regardless of what's there today.

Pin the ServiceMonitor and PrometheusRule from Section 8 at install time, not as an afterthought — import the same 4 alert pairs (`KarpenterClusterStateDesynced`, `KarpenterPodPending`, `KarpenterNoProvisioningActivity`, `KarpenterNodeClaimNotReady`) and the vendored Grafana dashboard from day one, matching the existing fleet.

### Step 3 — Interruption handling (do this even if you start on-demand-only)

This fleet currently skips this because it runs zero spot capacity (see Section 5) — but for a **new, prod-ready setup**, wire it up from the start regardless of your initial capacity-type choice, since it's cheap to set up now and expensive to retrofit once workloads depend on its absence:

1. Create an SQS queue named to match `settings.interruptionQueue` above (e.g. `<cluster-name>-karpenter`).
2. Add an EventBridge rule routing these events to that queue: `AWS Health Event` (for `EC2 Instance Rebalance Recommendation` and `EC2 Spot Instance Interruption Warning`), `EC2 Instance State-change Notification`, and `EC2 Instance Rebalance Recommendation`.
3. Grant the controller role `sqs:ReceiveMessage`/`DeleteMessage`/`GetQueueUrl`/`GetQueueAttributes` on that queue (already listed in Step 1).

With this in place, Karpenter gets ~2 minutes' advance notice on spot reclaim and can cordon/drain gracefully instead of losing the node abruptly — this is what makes it *safe* to adopt spot capacity later without a separate migration project.

### Step 4 — Default NodePool + EC2NodeClass

Start with one general-purpose NodePool, modeled on this org's own `dedicated-nodepool` pattern but simplified:

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023   # or Custom, per your Step 0 decision
  role: <node-role-name>
  subnetSelectorTerms:
    - tags: {karpenter.sh/discovery: <cluster-name>}
  securityGroupSelectorTerms:
    - tags: {karpenter.sh/discovery: <cluster-name>}
  tags:
    karpenter.sh/discovery: <cluster-name>
---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - {key: kubernetes.io/arch, operator: In, values: ["amd64"]}
        - {key: karpenter.sh/capacity-type, operator: In, values: ["on-demand"]}
        - {key: karpenter.k8s.aws/instance-category, operator: In, values: ["m", "r", "c"]}
        - {key: karpenter.k8s.aws/instance-generation, operator: Gt, values: ["4"]}
      nodeClassRef: {group: karpenter.k8s.aws, kind: EC2NodeClass, name: default}
      expireAfter: 720h   # 30d — forces periodic node refresh (AMI/patch currency),
                          # unlike this org's `Never`; recommended default for prod
      startupTaints:
        - {key: node.cilium.io/agent-not-ready, effect: NoExecute, value: "true"}
        # ^ ENABLE this if running Cilium — this org has it present but disabled
        #   everywhere (see Section 6); for a fresh prod setup, turn it ON.
  limits: {cpu: 1000, memory: 4000Gi}   # size to your actual capacity ceiling
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 5m
    budgets:
      - {nodes: "10%"}
      - {duration: 8h, nodes: "0", schedule: "0 9 * * mon-fri"}
        # example: no consolidation during a Monday-Friday business-hours window,
        # mirroring this org's own dedicated-nodepool pattern
```

Key production-readiness deltas from what this org currently runs:
- `expireAfter: 720h` instead of `Never` — periodic forced node replacement keeps AMI/kernel patches current; `Never` (as used in this org's `trino-arm64-nodepool`) is a deliberate tradeoff for workload stability that you should only copy if you have a good reason (e.g. long-running stateful jobs sensitive to node churn).
- `startupTaints` for Cilium **enabled**, not left commented out — closes the pod-scheduled-before-CNI-ready gap flagged in Section 6.
- A real business-hours consolidation budget from day one, not something bolted on after an incident.

### Step 5 — Protect critical workloads from disruption

Add `karpenter.sh/do-not-disrupt: "true"` as a pod annotation (not a NodePool-level setting) on anything that must never be evicted by consolidation — single-replica stateful workloads, long-running batch jobs, etc. This is a per-pod opt-out, so it needs to be part of your workload deployment templates, not something Karpenter config alone can enforce.

### Step 6 — Validate before trusting it in prod

1. Deploy a throwaway deployment requesting more CPU than any existing node group has free — confirm Karpenter provisions a new node within ~60-90 seconds and the pod schedules.
2. Scale that deployment to zero — confirm the node is consolidated away within `consolidateAfter` (5m in the example above), respecting the disruption budget.
3. Manually terminate a Karpenter-provisioned node via the EC2 console (or trigger a real spot interruption in a non-prod account) — confirm the interruption queue path drains it gracefully rather than the pod just disappearing.
4. Confirm `KarpenterClusterStateDesynced` and the other 3 alerts fire correctly by temporarily breaking the controller's IAM permissions in a test cluster and watching the alert trip — don't wait for a real prod incident to discover the alerting doesn't work.
5. Confirm the Grafana dashboard renders real data (node/pod distribution, nodepool usage %) before calling the rollout done.

### Step 7 — Rollback plan

Define before go-live: if Karpenter misbehaves in prod (e.g. runaway provisioning, wrong instance types, cost spike), the fastest safe mitigation is `kubectl scale deployment karpenter -n kube-system --replicas=0` — this stops all new provisioning/consolidation activity immediately without touching already-running nodes, buying time to fix the NodePool config before resuming. Keep the previous scaling mechanism (a manually-sized managed node group, or cluster-autoscaler if migrating from it) not fully torn down until Karpenter has run cleanly in prod for at least one full traffic cycle (including any weekly/monthly peak), so you have a fallback if a rollback is ever needed.

---

*Compiled from a read-only review of the adCloud monorepo (`cloud/kube-system/`, `argo/ArgoCD/`, vendored Helm chart, per-cluster `values.yaml`) — September 2026. Section 9 is guidance synthesized from the fleet's existing working configuration plus standard Karpenter production practices for gaps this fleet currently leaves open; it is not a transcription of a documented internal process.*
