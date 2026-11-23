# 📈 UNIT IV — PART 6: SCALING MONITORING SOLUTIONS

> **Covered in this document:** why monitoring itself must scale (growth in targets, series, retention, queries, teams) → the **scaling dimensions** and symptoms → a **capacity-planning method** with a worked example (series, samples/s, disk, RAM) → the **scaling ladder**: (1) optimize first (cardinality control, relabeling, sample limits, recording rules, query limits), (2) **high availability** (replica pairs, Alertmanager clustering), (3) **functional sharding** (per team/service/region), (4) **horizontal sharding** with `hashmod`, (5) **federation** (hierarchical, cross-service) and its limits, (6) **remote write/read** and **Agent mode**, (7) **long-term storage and global view**: **Thanos**, **Grafana Mimir/Cortex**, **VictoriaMetrics**, others (architectures, comparison, downsampling) → **Kubernetes monitoring** (kube-prometheus-stack, Prometheus Operator CRDs `ServiceMonitor`/`PodMonitor`/`PrometheusRule`, kube-state-metrics, cAdvisor, key queries and alerts) → dynamic environments and churn → **cloud/managed monitoring** (CloudWatch, Amazon Managed Prometheus/Grafana, Azure Monitor, Google Managed Prometheus; self-managed vs managed) → scaling Grafana and Alertmanager → **observability cost management** → **meta-monitoring** ("who watches the watcher") → security and multi-tenancy → decision table by symptom → common mistakes → viva → master sheet.

---

Absolutely mate. 😤🔥 So far one Prometheus + one Grafana monitors a few servers. Real companies (think Netflix and Uber from Unit I) monitor **thousands of hosts, millions of series and hundreds of teams**. Your syllabus line: **"Scaling monitoring solutions."**

---

# 1️⃣ WHY MONITORING MUST SCALE 🏔️

```
Unit I: autoscaling, microservices, containers → targets appear/disappear constantly, count grows 10-100×
More services × more labels = more time series (cardinality)
Longer retention (compliance, trend analysis, capacity planning) = more storage
More dashboards + alerts + engineers = more queries
More regions/clusters = need a GLOBAL view
Monitoring is production-critical: it must stay up when everything else is failing (HA)
```

## 1.1 Scaling dimensions

| Dimension | What grows | Symptom when it breaks |
|---|---|---|
| **Active series (cardinality)** | Memory in the head block | Out-of-memory crashes, slow restarts (WAL replay takes long) |
| **Ingestion rate (samples/s)** | CPU, disk I/O, network | Scrape lag/`scrape_duration_seconds` high, missed scrapes |
| **Query load** | CPU/memory during queries | Slow/timeouts in Grafana, "query processing would load too many samples" |
| **Retention** | Disk | Disk full; slow queries on large ranges |
| **Availability** | Single point of failure | Monitoring blind during the outage you most need it |
| **Geography/organization** | Clusters, regions, teams | No global view; noisy shared system; no tenant isolation |
| **Alerting/rules** | Rule evaluation time | Rule groups late, `rule_group_iterations_missed` |

## 1.2 Single Prometheus: how far does it go?

A single, well-sized Prometheus server can handle **millions of active series** and hundreds of thousands of samples per second on modern hardware. Do **not** scale out prematurely: first measure, then optimize, then split.

```
Golden rule: SCALE UP & OPTIMIZE first → then split by function → then add a global/long-term layer.
```

---

# 2️⃣ CAPACITY PLANNING ⭐⭐⭐

## 2.1 The formulas

```
samples per second     = active series ÷ scrape_interval (s)
disk (bytes)           = retention (s) × samples per second × bytes per sample (≈ 1-2)
RAM (rough)            ≈ active series × a few KB per series (commonly ~3-6 KB incl. overhead; measure your own!)  (+ headroom for queries and compaction)
```

## 2.2 Worked example 🧮

Fleet: **2,000 servers** × ~1,000 Node Exporter series each = **2,000,000 series**
Kubernetes: **20,000 pods** × ~50 series each = **1,000,000 series**
Total ≈ **3,000,000 active series**, scrape interval **30 s**, retention **15 days**.

```
samples/s = 3,000,000 ÷ 30                = 100,000 samples/second
15 days   = 15 × 86,400                    = 1,296,000 seconds
disk      ≈ 1,296,000 × 100,000 × 1.5 B   ≈ 194 GB        (add 20-30% headroom → ~250 GB SSD)
RAM       ≈ 3,000,000 × ~4 KB             ≈ 12 GB          (plan 2× for queries/compaction → ~24-32 GB node)
```

(Numbers are estimates: your compression ratio and per-series memory vary with label sizes and churn. Validate with real metrics, Section 2.3.)

## 2.3 Measure, don't guess: key self-metrics

```promql
prometheus_tsdb_head_series                                   # active series (memory driver)
rate(prometheus_tsdb_head_samples_appended_total[5m])         # ingestion rate (samples/s)
prometheus_tsdb_storage_blocks_bytes                          # disk used by blocks
process_resident_memory_bytes{job="prometheus"}               # actual RAM
rate(prometheus_tsdb_head_series_created_total[5m])           # series churn (high = ephemeral labels/pods)
prometheus_engine_query_duration_seconds                      # query latency
prometheus_rule_group_last_duration_seconds                   # rule evaluation time (must be < interval)
scrape_duration_seconds, scrape_samples_scraped               # per-target cost
```

Also: **Status → TSDB Status** page (top 10 label names/metric names by series count) to find the biggest offenders.

## 2.4 Cardinality budgeting

```
Series per metric ≈ product of distinct values of each label
http_requests_total{method(5) × route(50) × status(8) × pod(200)} = 400,000 series from ONE metric
Danger signs: labels like user_id, ip, url, request_id, trace_id, container_id churn, build IDs
```

---

# 3️⃣ STEP 1: OPTIMIZE BEFORE YOU SCALE OUT ✅

## 3.1 Cut cardinality at the source

```yaml
scrape_configs:
  - job_name: "k8s-pods"
    sample_limit: 50000          # fail the scrape if a target exposes more than this many samples (protects Prometheus)
    label_limit: 30              # max labels per series
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: "go_gc_.*|go_memstats_.*"          # drop rarely used runtime metrics
        action: drop
      - regex: "pod_template_hash|controller_revision_hash|uid"
        action: labeldrop                          # drop churn-heavy labels
      - source_labels: [__name__, le]
        regex: "http_request_duration_seconds_bucket;(0.001|0.002|0.003)"
        action: drop                               # drop unneeded histogram buckets
```

## 3.2 Other optimizations

| Technique | Benefit |
|---|---|
| Increase `scrape_interval` for non-critical jobs (60 s instead of 15 s) | Fewer samples/s |
| **Recording rules** for dashboards and alerts (Part 3) | Dashboards query small precomputed series instead of scanning thousands |
| Query limits: `--query.max-samples`, `--query.timeout`, `--query.max-concurrency` | Protect against heavy/accidental queries |
| Avoid `{__name__=~".+"}` and very long ranges in panels; sane dashboard refresh (30-60 s) | Lower query load |
| Fast local SSD/NVMe, enough RAM; keep data dir off NFS | I/O performance |
| Use **native histograms** where supported (fewer series than many classic buckets) | Lower cardinality for latency metrics |
| Drop unused exporters/collectors (`--no-collector.*` on node_exporter) | Smaller payloads |
| Set `--storage.tsdb.retention.size` as a safety cap | Avoid disk-full |
| Upgrade regularly (performance improvements land in most releases) | Free gains |

---

# 4️⃣ STEP 2: HIGH AVAILABILITY (HA) ⭐⭐

## 4.1 Prometheus HA = run **two identical replicas**

```
              ┌── Prometheus A (replica="A") ──┐                       ┌── Alertmanager 1 ◀─┐ gossip (:9094)
 Targets ◀────┤                                 ├──── alerts ────────▶ ├── Alertmanager 2 ◀─┤ cluster (dedup)
 (scraped by  └── Prometheus B (replica="B") ──┘  (BOTH send to ALL)   └── Alertmanager 3 ◀─┘
  both)              ▲ both are queried by Grafana/Thanos Querier (dedup by replica label)
```

Config differences: identical scrape/rules configs, different **external label**:

```yaml
global:
  external_labels:
    cluster: prod-mumbai
    replica: A          # B on the second server
```

Facts to know:

```
• Replicas are NOT byte-identical (scrape times differ slightly) → fine for monitoring, not for billing
• Both evaluate rules and fire alerts → Alertmanager DEDUPLICATES identical alerts (same labels, replica label excluded via alert_relabel_configs)
• Grafana can point at one replica (or a load balancer); for seamless global queries use Thanos Querier/Mimir, which dedup by `replica`
• Always tell Prometheus about ALL Alertmanager instances (do NOT put a load balancer in front of the Alertmanager cluster)
```

Drop the `replica` label from alerts so duplicates match:

```yaml
alerting:
  alert_relabel_configs:
    - regex: replica
      action: labeldrop
  alertmanagers:
    - static_configs:
        - targets: ["am1:9093", "am2:9093", "am3:9093"]
```

## 4.2 Alertmanager HA cluster

```bash
alertmanager --config.file=alertmanager.yml --cluster.listen-address=0.0.0.0:9094 --cluster.peer=am1:9094 --cluster.peer=am2:9094
```

Peers gossip **silences and notification logs**, so each notification is sent once even if all replicas receive the alert. Run an odd number (3) across failure domains/AZs (Unit I).

## 4.3 What HA does and doesn't give you

| Protects against | Doesn't protect against |
|---|---|
| A Prometheus process/host crash (other replica keeps scraping/alerting) | Loss of **both** replicas/the whole region |
| Rolling upgrades without blind spots | Long-term data loss (local disks) → remote write/Thanos |
| | Cardinality explosion (both replicas hit it) |

---

# 5️⃣ STEP 3: FUNCTIONAL (VERTICAL) SHARDING ⭐

Split **by purpose, team or environment**: each Prometheus owns a manageable slice.

```
Prometheus-infra   → node_exporter for all servers
Prometheus-apps-A  → services of team A          Prometheus-apps-B → services of team B
Prometheus-k8s-<cluster> → one per Kubernetes cluster     Prometheus-<region> → one per region/data center
```

Benefits: blast-radius isolation, per-team ownership and tuning, simple. Drawbacks: **no single place to query everything** (needs a global view layer: federation, Thanos, Mimir, or Grafana with many data sources).

Best practice: add `external_labels` (`cluster`, `region`, `env`, `team`) so data stays distinguishable when combined later.

---

# 6️⃣ STEP 4: HORIZONTAL SHARDING WITH `hashmod` ⭐

When **one job** (e.g., 10,000 pods) is too big for one server, split **targets** across shards:

```yaml
# shard 0 of 3 (change regex to ^1$ and ^2$ on the other two servers)
scrape_configs:
  - job_name: "big-fleet"
    file_sd_configs:
      - files: ["/etc/prometheus/targets/*.json"]
    relabel_configs:
      - source_labels: [__address__]
        modulus: 3
        target_label: __tmp_hash
        action: hashmod
      - source_labels: [__tmp_hash]
        regex: ^0$
        action: keep
```

Each shard scrapes one third of the targets. You then need a **query layer that fans out to all shards** (Thanos Querier, Mimir, or federation) because no single shard has all data. (The Prometheus Operator can shard automatically with `shards: N` in the Prometheus CR.)

---

# 7️⃣ STEP 5: FEDERATION ⭐⭐

> **Federation** lets one Prometheus **scrape selected series from another Prometheus** via its `/federate` endpoint.

## 7.1 Hierarchical federation (global view)

```
        ┌───────────── GLOBAL Prometheus (aggregates) ─────────────┐
        │ scrapes /federate from each regional server (only chosen, pre-aggregated series)
        ▼                       ▼                       ▼
  Prometheus Mumbai      Prometheus Frankfurt     Prometheus Virginia   (each scrapes its own region's targets in full detail)
```

```yaml
# on the GLOBAL Prometheus
scrape_configs:
  - job_name: "federate"
    scrape_interval: 30s
    honor_labels: true                         # keep original job/instance labels
    metrics_path: /federate
    params:
      "match[]":
        - '{__name__=~"job:.*"}'               # only recording-rule aggregates (job:...)
        - '{job="node"}'                       # (avoid pulling everything!)
    static_configs:
      - targets: ["prom-mumbai:9090", "prom-frankfurt:9090", "prom-virginia:9090"]
```

## 7.2 Cross-service federation

A Prometheus for service A scrapes a few metrics from service B's Prometheus (e.g., a shared database's metrics) so it can alert on cross-service conditions.

## 7.3 Limits of federation ⚠️

```
✘ Not meant to copy ALL data: the global server becomes the bottleneck (same cardinality problem, bigger)
✘ Introduces lag, extra scrape failures, and staleness/gaps
✘ No long-term storage benefit
✔ Good for: a SMALL set of aggregated series (recording rules like job:..., SLO ratios) for global dashboards and alerts
✔ For raw global views/long-term storage prefer remote write or Thanos/Mimir (next sections)
```

---

# 8️⃣ STEP 6: REMOTE WRITE, REMOTE READ AND AGENT MODE ⭐⭐

## 8.1 `remote_write`

Prometheus **streams samples** to a remote endpoint (long-term/scalable store) in near real time:

```yaml
remote_write:
  - url: "https://mimir.example.com/api/v1/push"       # Mimir/Cortex/Thanos Receive/VictoriaMetrics/AMP/Grafana Cloud
    basic_auth:
      username: tenant-1
      password_file: /etc/prometheus/remote_password
    queue_config:
      capacity: 10000
      max_shards: 30
      max_samples_per_send: 2000
    write_relabel_configs:                              # send only what you need (cost control)
      - source_labels: [__name__]
        regex: "node_.*|job:.*|up"
        action: keep
```

`remote_read` lets Prometheus query the remote store (used less often; Thanos/Mimir queriers handle reads). Prometheus 3.x also supports the newer **Remote Write 2.0** protocol (more efficient) with compatible receivers; check your backend's support.

## 8.2 Agent mode (lightweight collectors) ⭐

```bash
prometheus --enable-feature=agent ...        # older flag form; current docs use `--agent` in recent versions — check your release
```

**Agent mode** runs Prometheus as a **scraper + forwarder only** (no local querying/alerting/long storage; small WAL buffer) that remote-writes to a central store. Use at the edge/per cluster, with Mimir/Thanos Receive/AMP central. Alternatives with the same role: **Grafana Alloy**, **OpenTelemetry Collector**, **vmagent**.

```
 Edge cluster: [Agent: scrape → WAL buffer] ──remote_write──▶ CENTRAL (Mimir/Thanos/AMP/VM) ──▶ Grafana (global dashboards, rules, alerts)
```

---

# 9️⃣ STEP 7: LONG-TERM STORAGE AND GLOBAL VIEW ⭐⭐⭐

## 9.1 Thanos 🛰️ (add-on to existing Prometheus)

```
 Prometheus A + Sidecar ─┐   (sidecar uploads 2h blocks to object storage and serves recent data via StoreAPI)
 Prometheus B + Sidecar ─┤──▶ THANOS QUERIER (global PromQL; DEDUPLICATES HA replicas via replica label; fans out) ◀── Grafana
 Store Gateway ──────────┘     ▲  (serves historical blocks from S3/GCS/Azure Blob)
 Object storage (S3) ◀── Compactor (compacts + DOWNSAMPLES to 5m and 1h resolution, enforces retention)
 Optional: Ruler (global rules/alerts) · Query Frontend (query caching/splitting) · Receive (remote-write ingest) · Alertmanager as usual
```

| Component | Role |
|---|---|
| **Sidecar** | Runs next to Prometheus; uploads TSDB blocks to object storage; exposes StoreAPI for recent data |
| **Store Gateway** | Makes historical blocks in object storage queryable |
| **Querier** | One PromQL endpoint across everything; deduplicates replicas (`--query.replica-label=replica`) |
| **Compactor** | Compacts blocks, **downsamples** (raw → 5 m → 1 h), applies retention per resolution |
| **Ruler** | Evaluates rules against global data |
| **Receive** | Accepts `remote_write` (when Prometheus can't be sidecar'd, or multi-tenant ingest) |
| **Query Frontend** | Caching and splitting of long-range queries |

Why it's popular: **keeps your existing Prometheus**, adds **unlimited cheap retention (object storage)**, **global view**, **HA dedup**, **downsampling**.

Minimal sidecar example:

```bash
thanos sidecar --tsdb.path=/var/lib/prometheus --prometheus.url=http://localhost:9090 \
               --objstore.config-file=/etc/thanos/s3.yml --grpc-address=0.0.0.0:10901
# Prometheus must run with: --storage.tsdb.min-block-duration=2h --storage.tsdb.max-block-duration=2h and external_labels set
```

`s3.yml`:

```yaml
type: S3
config:
  bucket: "my-thanos-metrics"
  endpoint: "s3.ap-south-1.amazonaws.com"
  region: "ap-south-1"
  # use an IAM role on EC2/EKS instead of keys (Unit I/V)
```

## 9.2 Grafana Mimir (and Cortex) 🌐

> **Mimir** (successor/evolution of **Cortex**, by Grafana Labs, AGPL) is a **horizontally scalable, multi-tenant, long-term** Prometheus-compatible store fed by `remote_write`.

```
Prometheus/Agents ──remote_write──▶ DISTRIBUTOR ─▶ INGESTERS (in-memory + WAL, replicated) ─▶ object storage blocks
Grafana ──PromQL──▶ QUERY FRONTEND ─▶ QUERIER ─▶ (ingesters + STORE-GATEWAY) ; COMPACTOR ; RULER ; ALERTMANAGER (multi-tenant)
```

Key properties: **multi-tenancy** (tenant ID via `X-Scope-OrgID`), scales to **billions of series** with microservices deployment (or a simpler monolithic mode), high availability through replication, global rules/alerts, queryable with plain PromQL from Grafana.

## 9.3 VictoriaMetrics 🚀

Prometheus-compatible TSDB focused on **efficiency** (lower RAM/disk per series), simple single-binary mode and a cluster mode; components: `vmagent` (scraper/forwarder), `vmalert` (rules), `vmstorage/vminsert/vmselect` (cluster). Supports Prometheus `remote_write` and PromQL (plus MetricsQL extensions).

## 9.4 Other options

| System | Notes |
|---|---|
| **M3 (M3DB)** | Metrics platform created at **Uber** (Unit I case study) for massive scale; check current project activity before choosing |
| **InfluxDB / TimescaleDB** | Time-series databases with different query languages/models |
| **Amazon Managed Service for Prometheus (AMP)** | AWS-managed Prometheus-compatible backend (Section 12) |
| **Grafana Cloud** | Hosted Mimir/Loki/Tempo/Grafana |

## 9.5 Comparison table ⭐⭐⭐

| | **Plain Prometheus** | **Federation** | **Thanos** | **Mimir / Cortex** | **VictoriaMetrics** | **Managed (AMP, Grafana Cloud)** |
|---|---|---|---|---|---|---|
| Role | Collect + store + alert (single node) | Aggregate selected series | Global view + long-term on top of Prometheus | Scalable multi-tenant long-term store | Efficient Prometheus-compatible TSDB | Vendor operates it |
| Long-term storage | Local disk only | No | **Object storage (S3)** | **Object storage** | Local/cluster disks (efficient) | Included |
| Global query | No | Limited (aggregates) | **Yes (Querier)** | **Yes** | Yes (cluster) | Yes |
| HA dedup | Replica pair, manual | — | **Yes** | **Yes** | Yes | Yes |
| Multi-tenancy | No | No | Limited (Receive) | **Strong** | Cluster tenants | Yes |
| Downsampling | No | — | **Yes (5m/1h)** | Limited/no (cost via compression) | Yes (enterprise/options) | Varies |
| Ops complexity | Low | Low | Medium | High (microservices) / Medium (monolithic) | Low-Medium | **Lowest** |
| Ingest path | Scrape | Scrape `/federate` | Sidecar upload or Receive | `remote_write` | `remote_write`/scrape | `remote_write` |
| Best for | Small-mid, single cluster | Few aggregated series | Keep Prometheus, add retention + global view | Large SaaS-like, many tenants | Cost/efficiency-focused teams | Teams without ops capacity |

---

# 🔟 KUBERNETES MONITORING ⭐⭐⭐

Kubernetes (the platform from Unit I's orchestration section) is dynamic: pods churn, IPs change, autoscalers add nodes. Monitoring must follow.

## 10.1 What to monitor (layers)

| Layer | Source | Examples |
|---|---|---|
| **Nodes (hosts)** | node-exporter DaemonSet | CPU, memory, disk, network (USE) |
| **Containers** | cAdvisor (built into kubelet) | `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`, throttling |
| **Kubernetes objects** | **kube-state-metrics** | `kube_pod_status_phase`, `kube_deployment_status_replicas_available`, `kube_pod_container_status_restarts_total` |
| **Control plane** | API server, etcd, scheduler, controller-manager | request latency, etcd leader, WAL fsync |
| **Applications** | `/metrics` endpoints | RED metrics, business metrics |
| **Cluster autoscaler/HPA** | metrics + events | pending pods, scaling events |

## 10.2 kube-prometheus-stack (the standard install) ⭐

> A **Helm chart** that deploys Prometheus Operator + Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics + a large set of default **dashboards and alert rules**.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

kubectl -n monitoring get pods
kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80        # Grafana at http://localhost:3000
kubectl -n monitoring get secret monitoring-grafana -o jsonpath='{.data.admin-password}' | base64 -d   # admin password
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090
```

(Release names and service names depend on the `helm install` name; check `kubectl -n monitoring get svc`.)

## 10.3 Prometheus Operator CRDs: monitoring as Kubernetes objects ⭐⭐

| CRD | Purpose |
|---|---|
| **`Prometheus`** | Declares a Prometheus deployment (replicas, shards, retention, storage) |
| **`Alertmanager`** | Declares an Alertmanager cluster |
| **`ServiceMonitor`** | "Scrape the pods behind this **Service** with matching labels" |
| **`PodMonitor`** | "Scrape these **pods** directly" |
| **`PrometheusRule`** | Alerting/recording rules as a Kubernetes object |
| **`AlertmanagerConfig`** | Routing/receivers per namespace/team |
| **`Probe`** | Blackbox probes |

A team monitors its app simply by shipping a `ServiceMonitor` with its manifests:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: flask-app
  namespace: apps
  labels:
    release: monitoring            # must match the Prometheus selector in the chart values
spec:
  selector:
    matchLabels:
      app: flask-app               # Service labels
  namespaceSelector:
    matchNames: [apps]
  endpoints:
    - port: http                   # named port on the Service
      path: /metrics
      interval: 15s
```

`PrometheusRule` example:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: flask-app-rules
  namespace: apps
  labels:
    release: monitoring
spec:
  groups:
    - name: flask-app
      rules:
        - alert: FlaskHighErrorRate
          expr: sum(rate(myapp_http_requests_total{status=~"5.."}[5m])) / sum(rate(myapp_http_requests_total[5m])) > 0.05
          for: 5m
          labels: { severity: critical }
          annotations: { summary: "Flask app error ratio above 5%" }
```

## 10.4 Essential Kubernetes queries and alerts

```promql
# CPU usage per pod (cores)
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!="", container!="POD"}[5m]))

# memory working set per pod
sum by (namespace, pod) (container_memory_working_set_bytes{container!="", container!="POD"})

# CPU throttling ratio (limits too low)
sum by (namespace, pod) (rate(container_cpu_cfs_throttled_periods_total[5m])) / sum by (namespace, pod) (rate(container_cpu_cfs_periods_total[5m]))

# crash-looping pods: restarts in the last 15 minutes
increase(kube_pod_container_status_restarts_total[15m]) > 3

# pods stuck Pending
sum by (namespace) (kube_pod_status_phase{phase="Pending"}) > 0

# deployment not fully available
kube_deployment_spec_replicas != kube_deployment_status_replicas_available

# node not ready
kube_node_status_condition{condition="Ready", status="true"} == 0
```

## 10.5 Kubernetes-specific scaling pitfalls

```
• Pod churn creates many short-lived series (labels like pod, pod_template_hash) → high series_created rate; drop useless labels, shorter retention for high-churn jobs
• One Prometheus per cluster + remote write to a central store (Mimir/Thanos/AMP) = standard multi-cluster pattern
• Set resource requests/limits for Prometheus itself; use persistent volumes; shard with the Operator (shards: N) for huge clusters
• Use ServiceMonitor/PodMonitor discovery rather than static targets
```

---

# 1️⃣1️⃣ DYNAMIC AND CLOUD ENVIRONMENTS 🌩️

| Challenge | Approach |
|---|---|
| Autoscaling EC2/VMSS adds/removes hosts | **Service discovery**: `ec2_sd_configs`, `azure_sd_configs`, `file_sd` generated by Terraform/CloudFormation (Unit II) |
| Short-lived instances/jobs | Push via **Pushgateway**/remote write; textfile collector; keep metrics at the service level, not instance level |
| Spot/serverless (Lambda) | Pull impossible: use CloudWatch metrics (exporter) or push/OTel; custom metrics via EMF (Embedded Metric Format) |
| Multi-account/multi-region | One collector/agent per account/region + central store; consistent `external_labels` |
| Consistent tagging | Enforce tags (`env`, `team`, `service`) via IaC; map cloud tags to Prometheus labels through relabeling |

---

# 1️⃣2️⃣ CLOUD-NATIVE AND MANAGED MONITORING ⭐⭐

## 12.1 AWS

| Service | Role |
|---|---|
| **Amazon CloudWatch** | Metrics, logs (Logs Insights), alarms, dashboards, **Container Insights**, **Synthetics** (canaries), **Application Signals**, EventBridge integration; EC2 basic metrics every 5 min (detailed 1 min extra cost), memory/disk need the **CloudWatch agent** |
| **AWS X-Ray / ADOT (OpenTelemetry distro)** | Tracing; OpenTelemetry collection |
| **Amazon Managed Service for Prometheus (AMP)** | Serverless, Prometheus-compatible storage/query/ruler; ingest via `remote_write` (SigV4 auth) |
| **Amazon Managed Grafana (AMG)** | Managed Grafana with AWS SSO/IAM integration |
| **CloudTrail / Config / GuardDuty** | Audit/compliance/security monitoring (Unit V) |

`remote_write` to AMP (conceptual):

```yaml
remote_write:
  - url: https://aps-workspaces.ap-south-1.amazonaws.com/workspaces/ws-XXXX/api/v1/remote_write
    sigv4:
      region: ap-south-1                    # uses the IAM role of the instance/pod (no keys in config)
```

CloudWatch → Prometheus: `yet-another-cloudwatch-exporter` / `cloudwatch_exporter` (pull CloudWatch metrics for dashboards/alerts alongside Prometheus data).

## 12.2 Azure and Google Cloud

| Provider | Services |
|---|---|
| **Azure** | **Azure Monitor** (metrics, Log Analytics workspaces, Application Insights, alerts, Workbooks), **Azure Monitor managed service for Prometheus**, **Azure Managed Grafana**, Container Insights for AKS |
| **Google Cloud** | **Cloud Monitoring/Logging/Trace**, **Google Cloud Managed Service for Prometheus** (Prometheus-compatible, global scale) |

## 12.3 Self-managed vs managed ⭐

| | Self-managed (Prometheus + Thanos/Mimir + Grafana) | Managed (AMP/AMG, Grafana Cloud, Azure Managed Prometheus) | Cloud-native tools (CloudWatch, Azure Monitor) |
|---|---|---|---|
| Control/customization | Maximum | Medium | Limited to the platform |
| Ops burden | High (scaling, upgrades, HA, storage) | Low | Lowest |
| Cost model | Infra + people | Ingestion/series/query based | Per metric/GB/alarm/dashboard |
| Portability | High (open standards) | High (PromQL/remote write) | Low (vendor APIs) |
| Time to value | Slower | Fast | Fastest for that cloud's services |
| Multi-cloud/on-prem | Excellent | Good | Poor |
| Typical choice | Platform teams, cost-sensitive, multi-cloud | Teams wanting Prometheus without ops | Quick coverage of cloud resources; combine with Prometheus for apps |

Common real-world pattern: **CloudWatch for AWS service metrics + Prometheus/OTel for applications and Kubernetes + Grafana as the single pane of glass.**

---

# 1️⃣3️⃣ SCALING GRAFANA AND ALERTING 🔧

| Component | Scaling approach |
|---|---|
| **Grafana** | Run 2+ instances behind a load balancer with a **shared PostgreSQL/MySQL** database (not SQLite); store sessions/cache externally if needed; provision dashboards from Git; set query timeouts/limits; use Query Frontend/caching (Thanos/Mimir); keep dashboards efficient (fewer panels, recording rules) |
| **Alertmanager** | 3-node cluster, route by team/severity; templates in Git; `amtool` checks in CI |
| **Rules** | Spread rule groups across rulers/shards (Thanos Ruler/Mimir Ruler); keep each group's evaluation under its interval; alert on `rule_group_iterations_missed` |
| **Dashboards at org scale** | Folder structure + permissions per team; golden dashboards generated as code (Grafonnet/Terraform) |
| **Logs/traces backends** | Scale Loki/Tempo by splitting read/write paths and using object storage (Part 7) |

---

# 1️⃣4️⃣ OBSERVABILITY COST MANAGEMENT 💰 (LINKS TO UNIT VI)

Telemetry volume often becomes one of the **largest cloud bills** (especially with SaaS pricing per host/series/GB).

| Lever | How |
|---|---|
| **Control cardinality** | Label hygiene, drop high-churn labels, enforce series budgets per team |
| **Filter before storing** | `write_relabel_configs`, `metric_relabel_configs`, OTel Collector processors |
| **Tiered retention + downsampling** | Raw 15-30 d; 5-minute resolution for months; 1-hour for years (Thanos compactor) |
| **Sampling for traces** | Head-based (e.g., 5%) and **tail-based** (keep errors/slow traces) |
| **Log hygiene** | Drop debug logs in production, sample noisy logs, structured logs, shorter hot retention; archive to cheap object storage |
| **Right-size scrape intervals** | 60 s for non-critical, 15 s for critical |
| **Chargeback/showback** | Attribute telemetry cost to teams via tenant/team labels |
| **Review unused metrics/dashboards/alerts regularly** | Delete what nobody uses (Mimir/Grafana usage insights show unused metrics) |

Cost rule of thumb: **if no dashboard, alert or investigation uses a metric, don't store it.**

---

# 1️⃣5️⃣ META-MONITORING: WHO WATCHES THE WATCHER? 🪞⭐

Monitoring that fails silently is worse than none.

```
✔ Monitor Prometheus/Alertmanager/Grafana themselves (their /metrics): up, WAL errors, rule failures, notification failures, disk usage, series growth
✔ Watchdog (always-firing) alert → external dead-man's-switch service (e.g., Dead Man's Snitch/healthchecks.io-style, or a second Prometheus) pages you if it STOPS arriving
✔ A second, independent Prometheus (or cloud service) in a DIFFERENT failure domain scrapes the primary's health
✔ External synthetic checks from outside your network (black-box) for the monitoring UI/endpoint and for key user journeys
✔ Alert on: prometheus_notifications_dropped_total, rule_group_iterations_missed, prometheus_tsdb_head_series growth, disk < 20% free, WAL corruptions, scrape failures spike
✔ Back up Grafana DB/dashboards (Git) and keep alert configs in Git
```

---

# 1️⃣6️⃣ SECURITY AND MULTI-TENANCY IN SCALED SETUPS 🔐 (BRIDGE TO UNIT V)

```
• TLS between components (remote_write, Thanos gRPC, Mimir); authentication (basic auth/OAuth proxy/mTLS) on all endpoints
• Tenant isolation (Mimir tenants, Grafana orgs/folders/teams, per-team data sources) so teams see only their data
• Object-storage buckets private, encrypted, least-privilege IAM roles (not long-lived keys); lifecycle rules for retention
• Limit who can run expensive queries; set per-tenant limits (ingestion rate, series, query size)
• Secrets in secret managers; audit logs for dashboard/alert changes; keep components patched (CVEs are published for Prometheus/Grafana)
• Telemetry can contain PII/secrets: scrub logs/labels at the collector
```

---

# 1️⃣7️⃣ DECISION TABLE: WHAT TO DO WHEN? 🎯

| Symptom / Need | First action | If still not enough |
|---|---|---|
| Prometheus RAM too high / OOM | Find top series (TSDB Status), drop labels/metrics, add `sample_limit`, upgrade/resize | Functional sharding or hashmod sharding; Agent + central store |
| Slow dashboards | Recording rules, shorter ranges, `$__rate_interval`, reduce panel count | Query frontend/caching (Thanos/Mimir), more RAM/CPU |
| Need > 15-30 days retention cheaply | Increase retention + disk | **Thanos** or **Mimir** with object storage + downsampling |
| Need a global view across clusters/regions | Federation of aggregates (small) | **Thanos Querier** / **Mimir** / managed |
| Single point of failure | Two replicas + 3-node Alertmanager cluster | Global HA via Thanos/Mimir |
| Many teams, isolation needed | Separate Prometheus per team + Grafana folders/orgs | Mimir multi-tenancy |
| Kubernetes clusters | kube-prometheus-stack per cluster | Remote write to central store; Operator sharding |
| Short-lived/edge workloads | Pushgateway/textfile for batch; Agent mode at edge | OTel Collector/Alloy to central store |
| No ops capacity | Managed Prometheus/Grafana (AMP/AMG/Grafana Cloud) | Cloud-native monitoring for cloud services |
| Telemetry bill exploding | Cardinality/log/trace sampling cuts, retention tiers | Chargeback, SLO-driven telemetry budgets |

---

# 1️⃣8️⃣ COMMON MISTAKES 🚨

| Mistake | Impact | Fix |
|---|---|---|
| Scaling out before measuring | Complexity without need | Measure series, ingestion, query load first |
| Using federation to copy everything | Global server overloaded | Federate aggregates only; use Thanos/Mimir |
| Load balancer in front of Alertmanager peers | Broken gossip/dedup behavior | List every Alertmanager in each Prometheus |
| HA pair without distinct `replica` external label | Duplicate series/ambiguity in global stores | Add `replica: A/B`, dedup in Querier |
| Unbounded labels in Kubernetes (pod UIDs, random names) | Cardinality blow-up and churn | `labeldrop`, aggregate to workload-level |
| Local-disk-only retention for critical history | Data loss on disk failure | Remote write/Thanos to object storage |
| One giant Prometheus for the whole company | Blast radius, hard to tune | Functional sharding + global layer |
| SQLite Grafana with multiple instances | Inconsistent state | Shared PostgreSQL/MySQL |
| Ignoring meta-monitoring | Silent monitoring outage | Watchdog + external checks |
| Alerting rules evaluated against slow federated data | Late/missed alerts | Evaluate alerts close to the data (local Prometheus) |

---

# 1️⃣9️⃣ EXAM ANSWER TEMPLATE ✍️

**Q. How can monitoring be scaled for a large distributed system? Explain Thanos/federation/remote write.**

```
1. Intro: scale problems (series, ingestion, retention, availability, global view) and the principle "optimize first, then split".
2. Capacity planning formulas (samples/s, disk, RAM) + one number example.
3. Optimization: cardinality control, relabeling, sample limits, recording rules, query limits.
4. HA: two replicas with external_labels, Alertmanager cluster, deduplication.
5. Sharding: functional (per team/region) and hashmod horizontal sharding.
6. Federation (hierarchical) + its limitations.
7. Remote write + Agent mode → long-term stores: Thanos (sidecar, store, querier, compactor), Mimir/Cortex, VictoriaMetrics (small table).
8. Kubernetes: kube-prometheus-stack, ServiceMonitor, kube-state-metrics, cAdvisor.
9. Managed options (AMP/AMG, Azure/GCP managed Prometheus, CloudWatch) vs self-managed.
10. Cost control + meta-monitoring + security. Conclusion.
```

---

# 2️⃣0️⃣ VIVA QUESTIONS (PART 6) 🎯

**Q1. Why do monitoring systems need scaling?** More targets, series, retention, queries and teams; plus monitoring must stay highly available.

**Q2. How do you estimate Prometheus disk and memory needs?** Samples/s = series ÷ scrape interval; disk = retention × samples/s × ~1-2 bytes; RAM ≈ active series × a few KB.

**Q3. What is cardinality and how do you reduce it?** Number of unique series; drop high-churn/unbounded labels, drop unneeded metrics via relabeling, use `sample_limit`.

**Q4. How is Prometheus made highly available?** Run two identical replicas with a `replica` external label, send alerts to a clustered Alertmanager, and dedupe in a global query layer.

**Q5. What is federation and when to use it?** One Prometheus scraping selected series from another's `/federate`; use it for small sets of aggregated series, not for copying all data.

**Q6. What is `remote_write`?** Streaming samples from Prometheus to a remote long-term or scalable store (Mimir, Thanos Receive, VictoriaMetrics, AMP).

**Q7. What is Prometheus Agent mode?** A lightweight mode that scrapes and remote-writes without local querying or long storage.

**Q8. Explain Thanos architecture.** Sidecar uploads blocks to object storage; Store Gateway serves history; Querier gives a global, deduplicated PromQL view; Compactor compacts and downsamples; Ruler/Receive/Query Frontend optional.

**Q9. Thanos vs Mimir?** Thanos extends existing Prometheus servers with object-storage retention and global query; Mimir is a multi-tenant, horizontally scalable ingest-and-store backend fed by remote write.

**Q10. What is `hashmod` sharding?** Relabeling that hashes target addresses into N buckets so each Prometheus shard scrapes a subset.

**Q11. How do you monitor Kubernetes?** kube-prometheus-stack: node-exporter, cAdvisor, kube-state-metrics, control-plane metrics, ServiceMonitors/PodMonitors and PrometheusRules via the Operator.

**Q12. What do ServiceMonitor and PrometheusRule do?** ServiceMonitor tells the Operator which Services' pods to scrape; PrometheusRule defines alerting/recording rules as Kubernetes objects.

**Q13. Self-managed vs managed Prometheus?** Self-managed gives control and portability but needs ops effort; managed (AMP, Grafana Cloud) removes scaling/HA burden at a usage-based cost.

**Q14. How do you control observability costs?** Limit cardinality, filter before storage, downsample and tier retention, sample traces, reduce log volume, right-size scrape intervals and review unused telemetry.

**Q15. What is meta-monitoring?** Monitoring the monitoring system itself (Watchdog alerts, external checks, self-metrics) so failures aren't silent.

---

# 🧠 PART 6 MASTER SHEET

```
SCALE DIMENSIONS: series (RAM) · samples/s (CPU/IO) · queries · retention (disk) · availability · geography/teams · rules
CAPACITY: samples/s = series/scrape_interval ; disk = retention × samples/s × ~1.5 B ; RAM ≈ series × ~3-6 KB ; example: 3M series @30s, 15d → 100k samples/s, ~194 GB, ~12 GB RAM
LADDER: 1 optimize (drop labels/metrics, sample_limit, recording rules, query limits, scrape intervals)
        2 HA (2 replicas + external_labels replica, Alertmanager cluster :9094, list ALL AMs)
        3 functional sharding (team/service/region/cluster) 4 hashmod sharding 5 federation (aggregates only)
        6 remote_write + Agent mode 7 Thanos / Mimir(Cortex) / VictoriaMetrics / managed
THANOS: sidecar + store gateway + querier (dedup) + compactor (downsample 5m/1h) + ruler/receive/query-frontend ; object storage (S3)
MIMIR: remote_write → distributor → ingesters → object storage ; multi-tenant (X-Scope-OrgID) ; billions of series
KUBERNETES: kube-prometheus-stack (Operator) ; node-exporter + cAdvisor + kube-state-metrics ; CRDs ServiceMonitor/PodMonitor/PrometheusRule ; per-cluster Prometheus + central store
CLOUD: CloudWatch (+agent, Container Insights) ; AMP + AMG ; Azure Monitor/Managed Prometheus/Managed Grafana ; Google Managed Prometheus ; pattern: CloudWatch + Prometheus + Grafana
GRAFANA/AM SCALING: shared PostgreSQL, multiple instances ; AM 3-node cluster ; rulers sharded
COST: cardinality, filtering, downsampling/tiering, trace sampling, log hygiene, chargeback
META-MONITORING: self-metrics, Watchdog + external dead-man switch, independent second monitor, external synthetic checks
SECURITY: TLS/auth, tenant isolation, private encrypted buckets with IAM roles, per-tenant limits, scrub PII
```

---

## 🫡 PART 6 STATUS

You can size a Prometheus deployment, optimize cardinality, design HA and sharding, choose between federation, remote write, Thanos, Mimir and managed services, monitor Kubernetes with the Operator, control telemetry cost and set up meta-monitoring.

**STOP here.** Next up:

> **PART 7 — Advanced Observability Strategies (logs, traces, OpenTelemetry, correlation, SLO-based alerting, synthetic/RUM, eBPF, AIOps, chaos engineering) 🔭**
