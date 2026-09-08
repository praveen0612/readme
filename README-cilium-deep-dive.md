# Cilium — Architecture Deep-Dive & Production Rollout Guide

Adobe Ad Cloud Monorepo — CNI / Network Policy / Observability Review

## Table of Contents

- [1. Is Cilium Actually Deployed, and Where](#1-is-cilium-actually-deployed-and-where)
- [2. Core CNI Configuration](#2-core-cni-configuration)
- [3. Complete Traffic Flow](#3-complete-traffic-flow)
- [4. Network Policy — Plain K8s vs. Cilium](#4-network-policy--plain-k8s-vs-cilium)
- [5. Service Mesh Reality Check](#5-service-mesh-reality-check)
- [6. Ingress Relationship](#6-ingress-relationship)
- [7. Adoption History](#7-adoption-history)
- [8. Known Incident — Identity Exhaustion](#8-known-incident--identity-exhaustion)
- [9. Setting Up Cilium on a New Prod Cluster](#9-setting-up-cilium-on-a-new-prod-cluster)

---

## 1. Is Cilium Actually Deployed, and Where

**Yes** — Cilium `v1.16.5`, deployed via ArgoCD (`argo/ArgoCD/<cluster>/kube-system/cilium.yaml` → `cloud/kube-system/<cluster>/cilium`), image `quay.io/cilium/cilium:v1.16.5`.

Deployed **only on the 7 EKS-prefixed clusters**: `eks-mgmt`, `eks-usw2-prod`, `eks-usw2-dev`, `eks-us-east-1`, `eks-euw1-prod`, `eks-aws-lab`, `eks-dev`.

**Not deployed** on the other prod clusters in this repo — `k8s01-usw2-prod`, `k8s01-euw1-prod`, `k8s01-usw2-dev`, plain `us-east-1`, `mgmt`, `aws-lab`, `dev`. Those instead run an older, self-managed stack with explicit `flannel/` and `kube-proxy/` DaemonSet directories checked into the repo (consistent with a kops/kubeadm-style cluster, not EKS-managed). No AWS VPC CNI or Calico manifests were found anywhere in this repo on any cluster.

**Confirmed: no CNI chaining.** Every Cilium ConfigMap sets `cni-exclusive: "true"` and chaining mode is unset — Cilium fully owns `/etc/cni/net.d` on the clusters it's deployed to; it does not coexist with another CNI.

## 2. Core CNI Configuration

```yaml
kube-proxy-replacement: "false"
routing-mode: "tunnel"              # VXLAN overlay
ipam: "cluster-pool"                # Cilium manages its own pod CIDR ranges
ipv4-native-routing-cidr: 10.244.0.0/14
enable-l7-proxy: "true"
external-envoy-proxy: "true"
enable-hubble: "true"
mesh-auth-enabled: "true"
clustermesh-enable-endpoint-sync: "false"
clustermesh-enable-mcs-api: "false"
```

**Important correction to the usual "Cilium replaces kube-proxy" pitch:** `kube-proxy-replacement` is explicitly **disabled** here (confirmed by commit `feature/INF-18464 - Disable kube-proxy-replacement on cilium eks-dev`). Cilium's eBPF datapath is *not* doing Service routing in place of kube-proxy on these clusters — kube-proxy's iptables rules are still doing that job. Cilium runs its VXLAN overlay + identity-based policy engine + Hubble observability *alongside* kube-proxy, not instead of it.

It also uses **`cluster-pool` IPAM** — Cilium manages its own pod IP ranges independently of AWS ENIs — rather than the AWS VPC CNI's native ENI-based IPAM. Tradeoff: more flexible pod-IP allocation and no ENI/IP-per-node limits, but pod IPs are not directly routable in the VPC the way ENI-mode IPs are (hence the VXLAN overlay to bridge nodes).

## 3. Complete Traffic Flow

```
Pod A wants to talk to Pod B (different node)
        │
        ▼
cilium-agent (per-node DaemonSet, eBPF programs attached to veth/pod interfaces)
        │
        ├─► Looks up Pod B's Cilium "security identity"
        │   (identity = hash of pod labels, NOT the pod IP — this is
        │    the core Cilium concept: policy is identity-based, not IP-based)
        │
        ├─► Checks any CiliumNetworkPolicy / NetworkPolicy applicable
        │   to that identity pair (allow/deny decided in-kernel via eBPF,
        │   no iptables involved for policy enforcement itself)
        │
        ├─► If enable-l7-proxy applies to this traffic (an HTTP/gRPC/Kafka
        │   rule is present) → redirects through cilium-envoy, Cilium's
        │   sidecar-less L7 proxy, for inspection before allow/deny
        │
        └─► Packet is encapsulated in VXLAN (routing-mode: tunnel) and
            sent over the overlay network to Pod B's node
        │
        ▼
Every allowed/denied flow is exported to Hubble (enable-hubble: true)
        │
        ▼
Hubble Relay + Hubble UI (hubble.<cluster>.k8s.tubemogul.info)
  → real-time flow visibility: who talked to whom, allowed or dropped, L3/L4/L7

Meanwhile, in parallel:
Service (ClusterIP) traffic ──► kube-proxy iptables rules (UNCHANGED —
                                  kube-proxy-replacement is disabled)
```

## 4. Network Policy — Plain K8s vs. Cilium

Plain `kind: NetworkPolicy` is actually **dominant** in this repo: **768** instances vs. only **118** `CiliumNetworkPolicy`. Cilium is reached for specifically when its extra expressiveness is needed.

Real example (`cloud/ns-team-adcloud-search-algo/ethos11-prod-or2/network-policies.yaml`):
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
spec:
  egress:
  - toServices:
    - k8sService: {namespace: default, serviceName: kubernetes}
  endpointSelector:
    matchLabels: {use-k8s-api-access-policy: "true"}
```
`toServices` targets a **Service by name**, dynamically resolved to whatever backend pod IPs it currently has. Plain `NetworkPolicy` can only match pod/namespace label selectors or static CIDRs — it can't express "allow traffic to whatever's currently behind this Service" without hardcoding IPs that will go stale.

**No real L7 (HTTP method/path-based) policy was found in use anywhere.** `enable-l7-proxy` is turned on at the platform level, but no committed manifest actually exercises HTTP-aware rules — the capability is available, not exploited.

## 5. Service Mesh Reality Check

- `mesh-auth-enabled: "true"` — Cilium's SPIFFE-based mutual-authentication capability is turned on.
- `clustermesh-enable-endpoint-sync: "false"` / `clustermesh-enable-mcs-api: "false"` — **multi-cluster mesh is explicitly disabled.** Every cluster is fully independent; there's no cross-cluster service discovery.
- No Istio, Linkerd, or any other sidecar-based mesh exists anywhere in the repo.

**Verdict: Cilium is used here as CNI + identity-based policy engine + flow-visibility tool (Hubble) — not as a full service mesh.** The mesh-capable features (L7 proxy, mTLS) are enabled at the config level but not exercised for actual mesh-style traffic management today.

## 6. Ingress Relationship

Cilium's own Ingress/Gateway API support is **not used**. Hubble's own UI ingress uses `ingressClassName: nginx-internal` — the same ingress class backing other services reviewed earlier (e.g. Alertmanager). Actual ingress routing is handled by **nginx-ingress-internal** plus a separate **AWS Load Balancer Controller**, both living in `kube-system` alongside Cilium, independently of it.

## 7. Adoption History

Real, incremental rollout tracked via a Jira ticket series over roughly a year: `INF-17982` → `INF-18387` → `INF-18464` → `INF-18588` → `INF-19308` → `INF-19922`, cluster-by-cluster. First commits adding a `cilium/` directory: `eks-dev` (`feature/INF-18464`), `eks-usw2-dev` (`feature/INF-18580`).

**Open question, unresolved from git history:** it is **not possible to tell from this repo alone** whether these EKS clusters previously ran the default AWS VPC CNI addon (installed out-of-band via the EKS API, not tracked in GitOps) before Cilium was added, or whether Cilium was installed at cluster-creation time on fresh clusters. No prior CNI manifest was ever deleted in this repo's history for those cluster directories — but that's also consistent with the default AWS VPC CNI addon never having been GitOps-managed in the first place, so its removal (if it happened) simply wouldn't show up here.

**No internal migration runbook or checklist exists in this repo.** The only markdown files mentioning "cilium" are vendored upstream Helm chart READMEs (auto-generated `helm-docs` values tables), not an org-authored adoption guide.

## 8. Known Incident — Identity Exhaustion

A `CiliumIdentitySpaceNearExhaustion` PrometheusRule fires above 80% of Cilium's 16.7M identity space, with a description that literally says "risking a repeat of the identity exhaustion incident." The identity ConfigMap explicitly excludes labels like `batch.kubernetes.io/job-name` and `kuberhealthy-run-id` from identity computation — specifically to stop short-lived batch/health-check pods from churning through unique identities. This is direct evidence of a real past production outage caused by identity exhaustion, and a concrete tuning response to it.

---

## 9. Setting Up Cilium on a New Prod Cluster

**Read this section as a recommended plan built from the org's own working reference configuration (the 7 `eks-*` clusters), not as a documented internal procedure — none exists in this repo.** Swapping or introducing a CNI is one of the highest-blast-radius changes you can make to a live cluster (it can take down all pod networking cluster-wide if done wrong), so treat every step below as something to validate in a non-prod cluster first, and get a second reviewer on the actual Helm values diff before applying to prod.

### Step 0 — Decide: new cluster, or migrating a live one?

This is the most consequential decision and the repo's own history doesn't settle it cleanly (see Section 7).

- **If it's a brand-new cluster with no workloads yet**: this is the low-risk path. Install Cilium as the CNI at cluster-bootstrap time, before any application workloads are scheduled. Skip to Step 2.
- **If it's an existing live prod cluster currently on AWS VPC CNI**: this is a genuinely hazardous operation. Changing a running cluster's CNI generally requires either (a) a full node-by-node replacement (cordon → drain → terminate → let new nodes join already running the new CNI), which needs your workloads to tolerate rolling pod disruption cluster-wide, or (b) building a parallel cluster with Cilium and migrating workloads over (blue-green at the cluster level), which is slower but far safer. **Do not attempt an in-place CNI flip on a running prod cluster without a maintenance window, a tested rollback plan, and ideally a canary node group first.** Given no internal precedent for this exists in the repo, treat AWS's and Cilium's own official CNI-migration guides as required reading before proceeding, not just this document.

### Step 1 — Capacity and network planning

- Confirm the VPC CIDR has room for Cilium's overlay: the reference config uses `ipam: cluster-pool` with `ipv4-native-routing-cidr: 10.244.0.0/14` — a dedicated, non-overlapping range separate from the VPC's own CIDR, since pod IPs won't be directly VPC-routable under `cluster-pool` mode (unlike AWS VPC CNI's ENI mode). Pick a `/14` (or size appropriately for your expected pod count) that doesn't collide with any existing VPC, peered VPC, or on-prem range.
- Decide up front whether you actually want `kube-proxy-replacement: "false"` (matching every existing cluster) or `"true"`/`"strict"`. The org has consistently chosen `false` — keeping kube-proxy in place — across all 7 clusters. Deviating from that without a specific reason adds operational inconsistency; if you do want the eBPF kube-proxy replacement's performance benefit, treat it as a deliberate, separately-reviewed decision, not a default.

### Step 2 — Deploy via the existing GitOps pattern

Match the pattern already used for every other cluster rather than inventing a new one:
1. Create `cloud/kube-system/<new-cluster>/cilium/` with the Helm-rendered manifests, copying the ConfigMap values from an existing prod cluster (e.g. `eks-usw2-prod`) as the starting point — this keeps `kube-proxy-replacement`, `routing-mode`, `ipam`, `enable-hubble`, and `mesh-auth-enabled` consistent with the rest of the fleet.
2. Update `ipv4-native-routing-cidr` to the range chosen in Step 1 if it must differ from other clusters (it generally should, to avoid any cross-cluster/VPC-peering CIDR collision).
3. Add the corresponding ArgoCD `Application` manifest at `argo/ArgoCD/<new-cluster>/kube-system/cilium.yaml`, pointing at that path, following the same `directory: {recurse: true}` / `syncPolicy.automated.selfHeal: true` pattern used elsewhere in this repo.
4. Do **not** enable `syncPolicy.automated` for the very first sync on a new cluster — apply manually once, verify `cilium-agent` DaemonSet pods go Ready on every node, then flip on auto-sync/self-heal.

### Step 3 — Validate before trusting it

- Confirm `cilium status` (via `cilium-cli` or `kubectl exec` into an agent pod) reports `OK` for the datapath, IPAM, and controllers on every node.
- Deploy a throwaway test pod pair and confirm cross-node connectivity over the VXLAN overlay works before scheduling real workloads.
- Bring up Hubble Relay + Hubble UI immediately (it's cheap and safe) — it gives you live flow visibility to catch policy or connectivity problems early, well before you'd notice them any other way.
- Import the existing `CiliumIdentitySpaceNearExhaustion` PrometheusRule (and the identity-exclusion ConfigMap entries for `batch.kubernetes.io/job-name` / similar short-lived-job labels) from an existing cluster from day one — this is a lesson already paid for once in this org; don't re-learn it on the new cluster.

### Step 4 — Roll out network policy gradually

- Start with plain Kubernetes `NetworkPolicy` for anything that fits (this is what 87% of policies in the existing fleet use) — it's simpler and portable if Cilium is ever removed.
- Reach for `CiliumNetworkPolicy` specifically for `toServices`-style rules or anything needing dynamic Service-based matching, mirroring the one concrete pattern already proven in this repo.
- Leave `enable-l7-proxy` on (matches the rest of the fleet) but don't feel obligated to write L7 policies just because the capability exists — no cluster in this org actually uses that today.

### Step 5 — Rollback plan

Define this **before** touching prod, not after something breaks:
- For a fresh cluster: rollback is simple — delete the cluster/node group and recreate, since no workloads are at risk yet.
- For an in-place migration on a live cluster: your rollback plan needs to be "revert nodes to the previous CNI via the same node-replacement mechanism used to roll forward," which means you need the previous CNI's manifests/config still available and tested, and a way to drain/replace nodes back. If you can't articulate this step concretely before starting, that's a signal you're not ready to do an in-place migration yet — building a parallel cluster and migrating workloads is the safer default in that case.

---

*Compiled from a read-only review of the adCloud monorepo (`cloud/kube-system/`, `argo/ArgoCD/`, chart configs, git history) — September 2026. Section 9 is guidance synthesized from the fleet's existing working configuration; it is not a transcription of a documented internal process, since none exists in-repo.*
