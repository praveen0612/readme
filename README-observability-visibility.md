# Application & Infra Visibility — Monitoring, Metrics, Alerting

Adobe Ad Cloud Monorepo — Observability Architecture Review

## Table of Contents

- [1. Architecture Overview](#1-architecture-overview)
- [2. Infra-Level Monitoring Stack](#2-infra-level-monitoring-stack)
- [2.1 End-to-End Pipeline Diagram](#21-end-to-end-pipeline-diagram)
- [3. Application-Level Observability](#3-application-level-observability--two-generations)
- [4. How Metrics Actually Get Scraped](#4-how-metrics-actually-get-scraped)
- [5. Why Multiple Pod Replicas Don't Produce Duplicate Metrics](#5-why-multiple-pod-replicas-dont-produce-duplicate-metrics)
- [6. HA Prometheus Replicas + Thanos Dedup](#6-ha-prometheus-replicas--thanos-dedup-a-different-problem)
- [6.1 Thanos ↔ S3 Long-Term Storage Flow](#61-thanos--s3-long-term-storage-flow)
- [7. Alerting & On-Call Routing](#7-alerting--on-call-routing)
- [8. Dashboards](#8-dashboards)
- [9. Known Gaps / Risks](#9-known-gaps--risks)
- [9.1 Pushgateway Collision Risk — Diagram](#91-pushgateway-collision-risk--diagram)

---

## 1. Architecture Overview

- Repo uses **GitOps**: manifests rendered (Bazel/Helm) → committed to `cloud/<namespace>/<cluster>/` → **ArgoCD** syncs into clusters (`selfHeal: true`).
- Deployment tracking = git history/PRs, not a separate dashboard (`docs/gitops_and_deployment.md`, the `kubedobe_deploy` pipeline).
- Visibility has 3 layers, each duplicated **per-cluster** rather than centralized: Infra monitoring → App instrumentation → Alert routing.

## 2. Infra-Level Monitoring Stack

| Component | Role | Location / Evidence |
|---|---|---|
| **Prometheus Operator** | Watches CRDs (`Prometheus`, `ServiceMonitor`, `PodMonitor`, `PrometheusRule`, `Alertmanager`). Renders `prometheus.yml` + rule files into a Secret/ConfigMap. **Never scrapes anything itself.** | `cloud/monitoring/<cluster>/adcloud-prometheus-operator` |
| **Prometheus** | The only component that actually scrapes. Runs k8s service discovery, pulls `/metrics`, evaluates alert rules. | `cloud/monitoring/<cluster>/adcloud-prometheus` |
| **config-reloader sidecar** | Watches mounted config file, hits `/-/reload` on change | Standard in every Prometheus StatefulSet |
| **Thanos** (sidecar/querier/store-gateway/compactor) | Long-term storage + merges results across redundant Prometheus replicas | `cloud/ops-prod/eks-mgmt/thanos-querier.yaml` |
| **Grafana** | Dashboards, LDAP-based auth | `cloud/ops-prod/eks-mgmt/grafana.yaml`, generated via Bazel target `//ops/grafana:eks-mgmt-grafana.gitops` |
| **CloudWatch exporter** | Bridges AWS CloudWatch metrics into Prometheus | `adcloud-cloudwatch-exporter` |
| **node-exporter / node-problem-detector / kube-state-metrics** | Node & cluster-level metrics | `cloud/monitoring/<cluster>/adcloud-prometheus-node-exporter` |
| **Karpenter** | Ships its own PrometheusRule for autoscaler health | `cloud/kube-system/eks-mgmt/karpenter/templates/prometheusrule.yaml` |

> **Note:** No Datadog used for core infra. PagerDuty is not used at the infra-monitoring layer — see [Alerting](#7-alerting--on-call-routing).

## 2.1 End-to-End Pipeline Diagram

```
┌─────────────┐         ┌──────────────────┐        ┌────────────────────┐
│  App Pod(s) │ exposes │  Kubernetes API   │ watched│ Prometheus Operator│
│ /metrics    │◄────────│ (Service/Pod objs)│◄───────│ (watches CRDs only)│
└─────────────┘  passive└──────────────────┘        └─────────┬──────────┘
      ▲  (never pushes)                                        │
      │                                     watches CRDs:       │ renders
      │                              ServiceMonitor/PodMonitor/  │
      │                              PrometheusRule/Alertmanager │
      │                                                          ▼
      │                                              ┌───────────────────────┐
      │                                              │ Generated prometheus.yml│
      │                                              │  + rule files (Secret) │
      │                                              └───────────┬───────────┘
      │                                                          │ mounted into
      │                                                          ▼
      │                                        ┌──────────────────────────────┐
      │                                        │  Prometheus Pod              │
      │            PULL /metrics on interval   │  ┌─────────────────────────┐ │
      └────────────────────────────────────────┼──┤ config-reloader sidecar │ │
                                                 │  │ (watches file, hits    │ │
                                                 │  │  /-/reload on change)  │ │
                                                 │  └─────────────────────────┘ │
                                                 │  ┌─────────────────────────┐ │
                                                 │  │ Prometheus server       │ │
                                                 │  │ - runs k8s SD           │ │
                                                 │  │ - SCRAPES targets       │ │
                                                 │  │ - evaluates alert rules │ │
                                                 │  └─────────────────────────┘ │
                                                 └──────────────┬───────────────┘
                                                                │ alerts fire
                                                                ▼
                                                    ┌───────────────────────┐
                                                    │     Alertmanager      │
                                                    │  routes by label match│
                                                    └──┬────────┬────────┬──┘
                                                       ▼        ▼        ▼
                                                    Slack   PagerDuty  Nexus/
                                                                       DeadMan's
```

Key point: the **Operator never scrapes anything** — it's a config compiler. **Prometheus** is the only thing that pulls data.

## 3. Application-Level Observability — Two Generations

**Legacy Java/Spring services** (`actv/`, `stats/`, `search/`):
- Logback text logs + homegrown correlation ID (`RequestIdFilter.java` sets MDC field `requestId`, not W3C trace-context)
- Metrics via Spring Boot Actuator, not Micrometer or Datadog

**Modern Python AI-agent services** (`api/diagnostics_agent/*`):
- JSON logs → Splunk
- Native Prometheus `/metrics` endpoint
- Real OpenTelemetry tracing → Tempo, via `cloud/apps-prod/us-east-1/otel-collector.yaml`

**Example scrape wiring** (`search-data-api`):

```yaml
management.endpoints.web.exposure.include: health,info,actuator,prometheus
management.metrics.export.prometheus.enabled: true
```

```yaml
# deployment.yaml annotations
prometheus.io/scrape: "true"
prometheus.io/path: /actuator/prometheus
prometheus.io/port: "8181"
```

> **Gap:** old and new services can't be correlated end-to-end — different correlation-ID schemes, and only new services have tracing.

## 4. How Metrics Actually Get Scraped

Confirmed fact: `PodMonitor` exists only as a CRD definition in this repo — **zero real usage found**. Two mechanisms are actually used:

### Mechanism A — ServiceMonitor (Service → Endpoints indirection)

```yaml
kind: ServiceMonitor
spec:
  selector: {matchLabels: {app: event-exporter}}
  endpoints:
  - port: http-metrics
    metricRelabelings:
    - sourceLabels: [__name__]
      regex: 'kubernetes_build_info'
      action: drop
```

The Operator resolves `selector` → Service → its `Endpoints` object (one entry per backing pod) → writes a scrape job using `kubernetes_sd_configs: {role: endpoints}`.

### Mechanism B — raw additionalScrapeConfigs (more common here, bypasses Service)

```yaml
- job_name: "kubernetes-pods"
  kubernetes_sd_configs:
    - role: pod
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
    - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: ${1}:${2}
      target_label: __address__
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)
    - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: instance
  metric_relabel_configs:
    - source_labels: [category]
      regex: NBR_BAD_REQUEST
      action: drop
```

| Stage | Config Block | Timing | Purpose |
|---|---|---|---|
| Discover | `kubernetes_sd_configs` | pre-scrape | Enumerate every pod as a candidate target |
| Filter | `relabel_configs` (keep rule) | pre-scrape | Keep only pods annotated `prometheus.io/scrape: "true"` |
| Build URL | `relabel_configs` | pre-scrape | Construct scrape path/port from annotations |
| **Set identity** | `pod_name → instance` | pre-scrape | Guarantees a unique identity per replica |
| Prune noise | `metric_relabel_configs` | **post-scrape** | Drop/rename already-pulled samples (cardinality control) |

**Key distinction:** `relabel_configs` decide **which targets** get scraped and how (pre-scrape, works on `__meta_*` Kubernetes metadata). `metric_relabel_configs` decide **which samples** get kept (post-scrape, works on actual returned metric data).

## 5. Why Multiple Pod Replicas Don't Produce Duplicate Metrics

Prometheus is **pull-based** — apps never push, so there's no path for accidental duplicate sends.

```
3 replicas of a Deployment
  → role: pod SD enumerates each pod individually
  → each target's __address__ = <unique pod IP>:<port>
  → instance label explicitly set to pod name/IP
  → 3 DISTINCT time series:
       metric{instance="10.x.11:9404", job="X"} = 5
       metric{instance="10.x.12:9404", job="X"} = 3
       metric{instance="10.x.13:9404", job="X"} = 7
  → sum(metric{job="X"}) = 15
     (safe: summing 3 distinct series, not double-counting one)
```

No shared key exists for two pods to collide on — duplication is **structurally impossible** at this layer, not just handled after the fact.

## 6. HA Prometheus Replicas + Thanos Dedup (a Different Problem)

Don't confuse this with #5. Some clusters run `replicas: 2` Prometheus servers for HA — **both scrape the same pods independently**, producing 2 full copies of every series (distinguished by a `prometheus_replica` label).

**Topology:**

```
prometheus.eks-mgmt.k8s.tubemogul.info  (Ingress)
        ↓
Service "prometheus-operator-prometheus"  (ClusterIP)
        ↓
kube-proxy round-robins each connection to ONE of the 2 replica pods
```

**Correction to a common assumption:** Hitting the raw Prometheus ingress does **not** give you duplicate data — a ClusterIP Service sends your query to exactly one pod, so you get that replica's own local view (2-day retention here), which may be **incomplete** if that pod restarted/missed scrapes, but never duplicated.

The actual merge/dedup point is **Thanos Querier**:

```
thanos query --store=thanos-sidecar-0... --store=thanos-sidecar-1...
             --query.replica-label=prometheus_replica
```

Thanos Querier fans one query out to **both** sidecars and merges — without `--query.replica-label` you'd see 2 near-duplicate series; the flag collapses them and back-fills gaps from whichever replica has better coverage.

**Summary:** Raw Prometheus = single-replica, possibly-incomplete view. Thanos = merged, deduplicated, gap-filled, long-retention view.

## 6.1 Thanos ↔ S3 Long-Term Storage Flow

**Shared objstore config**, used by every Thanos component (`cloud/monitoring/eks-mgmt/adcloud-prometheus/templates/thanos-sidecar/secrets.yaml`):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: thanos-objstore-config
stringData:
  thanos-objstore-config.yaml: |
    type: s3
    config:
      bucket: adcloud-kubernetes-thanos
      endpoint: s3.us-east-1.amazonaws.com
      aws_sdk_auth: true
```

`aws_sdk_auth: true` means no static access keys are stored — every Thanos pod carries an `iam.amazonaws.com/role: ...k8s-monitoring-thanos-eks-mgmt` annotation, picked up by **kube2iam** (older node-agent pattern, not IRSA), granting temporary IAM credentials scoped only to `arn:aws:s3:::adcloud-kubernetes-thanos*`.

**Write path — Prometheus → Sidecar → S3:**

The `Prometheus` CR sets `retention: "2d"` (local disk only ever holds 2 days) and has a `thanos:` block pointing at the Secret above, which makes the Operator inject a **Thanos Sidecar container into the same pod as Prometheus**.

```
Prometheus writes 2-hour TSDB blocks to local disk
        │
        ▼ (once a block is sealed / no longer being written to)
Thanos Sidecar uploads the block ──────────► s3://adcloud-kubernetes-thanos/
        │
        ▼
Sidecar ALSO exposes a Store API for that pod's own data
(this is why Thanos Querier can talk to sidecars directly for "recent" data)
```

**Compaction/downsampling in S3** (`cloud/monitoring/eks-mgmt/adcloud-thanos/templates/compactor/statefulset.yaml`):

```
compact --objstore.config=$(OBJSTORE_CONFIG)
        --retention.resolution-raw=30d
        --retention.resolution-5m=120d
        --retention.resolution-1h=3y
```

The Compactor works directly against the S3 bucket (with a scratch PVC for local work): it merges many small 2-hour blocks into bigger ones, and produces downsampled 5-minute and 1-hour resolution copies so multi-month/year queries stay fast. These retention flags control *query resolution*, not deletion — raw full-resolution data is pruned after 30 days, 5m after 120 days, 1h kept for 3 years.

**Read path — Store Gateway → Querier** (`cloud/monitoring/eks-mgmt/adcloud-thanos/templates/store-gateway/statefulset.yaml`):

```
store --objstore.config=$(OBJSTORE_CONFIG) --min-time=-1y
```

Store Gateway indexes the bucket's block metadata (without downloading full blocks) and exposes everything back to 1 year via gRPC Store API — this is what lets you query data older than Prometheus's local 2-day retention. Thanos Querier fans a single query across **both**: one `--store=` flag at the store-gateway (S3/historical), and one `--store=` flag per Prometheus sidecar across the whole fleet (~30 of them, live/recent).

```
                         ┌──────────────────┐
   Live/recent data ◄────┤ Thanos Sidecars   │ (one per Prometheus replica, per cluster)
        │                └──────────────────┘
        │
   ┌────▼─────────┐
   │Thanos Querier │──── merges everything into one response
   └────┬─────────┘
        │
   Historical data ◄──┌──────────────────┐        ┌─────────────────┐
        │             │ Thanos Store GW  │◄───────┤   S3 bucket      │
        └─────────────┤ (reads index +   │        │ adcloud-         │
                       │  block metadata) │        │ kubernetes-thanos│
                       └──────────────────┘        └────────▲────────┘
                                                              │
                                                    ┌─────────┴────────┐
                                                    │ Thanos Compactor │
                                                    │ merges + downsamples
                                                    │ (30d raw / 120d 5m / 3y 1h)
                                                    └──────────────────┘
```

**The bucket itself** (`infrastructure/kubernetes/terraform/aws/k8s-thanons-s3-bucket/s3.tf`): `adcloud-kubernetes-thanos` (prod) and `adcloud-kubernetes-thanos-dev`, with S3 Intelligent-Tiering enabled and `prevent_destroy = true`. **No object-expiration lifecycle rule exists** — objects are versioned and kept indefinitely (only incomplete multipart uploads auto-abort after 7 days). So the compactor's retention flags control what you can query *efficiently*, but raw S3 objects are never actually deleted.

> **Gap worth flagging:** unbounded S3 growth/cost has not been reviewed anywhere found in-repo — no lifecycle expiration rule and no storage-size alerting were located.

## 7. Alerting & On-Call Routing

Every cluster's Alertmanager fans out to up to 3 channels based on severity/team labels:

```yaml
route:
  routes:
    - receiver: deadmanssnitch     # heartbeat — pages if Prometheus/AM itself dies
      match: {alertname: Watchdog}
    - receiver: nexus              # Adobe-internal ITSM/notification gateway
      match_re: {severity: ^(low|critical|critical_businesshours)$}
    - receiver: slack-k8s-alerts
      match_re: {severity: ^(slack|warning|low|critical|critical_businesshours)$}
```

- **PagerDuty** only wired for RTB, UDB, and some stats-prod clusters (e.g. `team: udb → udb-pagerduty`). Most other services (apps-prod, data-pipeline-prod, honeycomb, search) only get Slack + "nexus" — **no page, just a Slack message**.
- **ArgoCD notifications** installed everywhere but unconfigured (empty triggers/templates) — **sync failures aren't alerted anywhere**.
- CI/build failures go through a separate Argo Workflows path (`argo/*/configmap/slack_alert.py`) straight to Slack.
- Alert annotations link to runbooks: `Runbook: {{ .Annotations.runbook_url }}` → `rtb/runbooks/*.md`.

## 8. Dashboards

- **Grafana** dominates (371 repo hits vs. 275 New Relic, ~0 Datadog), but dashboard JSON isn't stored as code here — only app config (auth, data sources) is.
- **Kubernetes Dashboard** (cluster web UI, separate from Grafana) mid-migration: v1 (client-cert auth) → v2 (Dex + oauth2-proxy SSO), ArgoCD-driven.

## 9. Known Gaps / Risks

| # | Gap | Detail |
|---|---|---|
| 1 | **Plaintext Slack webhook** | Committed in every cluster's `alertmanager/secrets.yaml` — named "Secret" but not actually secret-managed |
| 2 | **No PagerDuty coverage** | Most services outside RTB/UDB/stats only alert to Slack, never page |
| 3 | **ArgoCD sync-failure alerting** | Installed but not configured — no alerts on sync failures |
| 4 | **No Airflow DAG-level failure alerting** | Only cleanup-oriented `on_failure_callback`s found, no Slack/email |
| 5 | **Pushgateway grouping-key collision risk** | Every pusher uses the same static grouping key (`job=pint_job`); stale pushed metrics never auto-expire. This is the **one place** true metric collision/staleness is a real, unmitigated risk — everywhere else, scrape-based target identity structurally prevents it |

## 9.1 Pushgateway Collision Risk — Diagram

```
 Batch Job Pod 1 ──┐
 Batch Job Pod 2 ──┼── pushadd_to_gateway(job="pint_job", ...) ──► Pushgateway
 Batch Job Pod 3 ──┘        (SAME static grouping key           (holds last-pushed
                              from every pod, every job)          value per label
                                                                   set, FOREVER —
                                                                   never expires)
                                                                        │
                                                                   scraped by
                                                                   Prometheus
                                                                   as if it were
                                                                   a live target
```

Source: `pint/platform/monitoring/prometheus_pusher.py`. `pushadd_to_gateway` only overwrites series with an identical metric-name + label-set combination, so it's mostly safe as long as label values (`job_name`, `status`, `start_time`) differ between concurrent pushes — but nothing structurally prevents a collision the way pod-IP-based scraping does everywhere else in this repo, and nothing ever auto-clears stale series.

---

*Compiled from a read-only review of the adCloud monorepo (`cloud/`, `ops/`, `argo/`, app source trees) — September 2026.*
