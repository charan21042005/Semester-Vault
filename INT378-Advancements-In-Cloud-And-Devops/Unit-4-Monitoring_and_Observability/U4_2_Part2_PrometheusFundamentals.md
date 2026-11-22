# 🔥 UNIT IV — PART 2: PROMETHEUS FUNDAMENTALS

> **Covered in this document:** what Prometheus is (history, CNCF, philosophy, what it is *not*) → **architecture** (server, retrieval, TSDB, PromQL engine, exporters, client libraries, Pushgateway, service discovery, Alertmanager, Grafana, remote storage) → the **data model** (metric names, labels, samples, time series, naming conventions, cardinality) → the **four metric types** (counter, gauge, histogram, summary) → the text **exposition format** and `/metrics` → jobs, instances and the automatic `up` metric → **configuration** (`prometheus.yml` section by section, static vs dynamic service discovery, relabeling, reloading, `promtool`) → installation overview (binary, systemd, Docker) and important flags → the web UI tour → the **TSDB** (WAL, head, blocks, compaction, retention, disk sizing formula) → **exporters** (Node Exporter in depth with metric names, Blackbox, cAdvisor, database exporters, textfile collector) → **instrumenting your own Python/Flask app** with `prometheus_client` (complete code) → Pushgateway (when and when not) → security → limitations → one scrape cycle dry run → common mistakes → viva → master sheet.

---

Absolutely mate. 😤🔥 Part 1 gave the theory of monitoring and observability. Part 2 starts the **tool that collects the metrics: Prometheus**, the heart of Practical 8.

---

# 1️⃣ WHAT IS PROMETHEUS? ⭐

> **Prometheus** is an open-source **monitoring and alerting toolkit** built around a **multi-dimensional time-series database**, a **pull-based collection model** over HTTP and a powerful query language (**PromQL**).

Key facts:

```
Origin        → built at SoundCloud (2012), open-sourced; inspired by Google's Borgmon
CNCF          → joined the Cloud Native Computing Foundation in 2016 (second project after Kubernetes); graduated in 2018
Language      → written in Go; a single static binary
Data model    → time series identified by metric name + key/value labels
Collection    → PULL: Prometheus scrapes HTTP endpoints (/metrics) at intervals
Query         → PromQL (Part 3)
Alerting      → alert rules in Prometheus + Alertmanager for routing/notifications (Part 4)
Storage       → local on-disk TSDB (single node, not clustered); remote write for long-term storage (Part 6)
Releases      → a new minor roughly every 6 weeks, plus Long Term Support (LTS) releases (check prometheus.io/download for the current ones)
```

Prometheus 3.0 (released Nov 2024) was the first major version in years: new UI, UTF-8 metric/label names, OpenTelemetry (OTLP) ingestion support and cleanup of old APIs. Releases since then continue on the 3.x line.

## 1.1 Philosophy (why it became the standard)

```
✔ Simple operation: one binary, no external dependencies
✔ Autonomous: each server is independent (no distributed consensus to break during an outage)
✔ Dimensional data: labels let you slice by instance, job, region, version, status code...
✔ Powerful queries: PromQL computes rates, percentiles, ratios on the fly
✔ Built for dynamic cloud environments: service discovery for Kubernetes, EC2, Consul, DNS
✔ Reliable when everything else is on fire: designed to be the monitoring system you can trust during an outage
```

## 1.2 What Prometheus is NOT 🚫

| It is not | Use instead |
|---|---|
| A log system | Loki, ELK |
| A tracing system | Jaeger, Tempo |
| Long-term, infinitely scalable storage by itself | Thanos, Mimir, VictoriaMetrics, remote write (Part 6) |
| A 100%-accurate billing/accounting system | Metrics are sampled; use an event store for money |
| A dashboard tool | **Grafana** (Part 5) |
| Clustered/HA out of the box | Run HA pairs + Thanos/Mimir (Part 6) |

---

# 2️⃣ PROMETHEUS ARCHITECTURE ⭐⭐⭐ (DRAW THIS)

```
                                    ┌─────────────────────────────────────────────────────────────┐
  Short-lived jobs ─push─▶ PUSHGATEWAY ─────┐                PROMETHEUS SERVER                      │
                                             │   ┌───────────────┐   ┌──────────────┐   ┌─────────┐ │
  Apps with client libs  ─/metrics──┐        └──▶│  RETRIEVAL    │──▶│  TSDB        │◀──│ PromQL  │ │◀─ queries ── GRAFANA
  (Python/Go/Java)                  │   scrape   │ (scraper)     │   │ (head + WAL  │   │ engine  │ │   (HTTP API)    API clients
  Node Exporter (host metrics) ─────┤  (HTTP     │      ▲        │   │  + blocks)   │   └─────────┘ │
  Blackbox exporter (probes) ───────┤   pull)    └──────┼────────┘   └──────┬───────┘        │       │
  cAdvisor, DB exporters ... ───────┘                   │                   │                ▼       │
                                         SERVICE DISCOVERY            RULES ENGINE ────alerts───▶ ALERTMANAGER ──▶ Email / Slack /
                                  (static, file, EC2, Kubernetes,     (recording + alerting rules)               PagerDuty / webhook
                                   DNS, Consul)                                                      
                                                                      REMOTE WRITE ─────────────▶ Long-term storage (Thanos/Mimir/VM)
                                    └─────────────────────────────────────────────────────────────┘
```

## 2.1 Components

| Component | Role |
|---|---|
| **Prometheus server** | Scrapes targets, stores time series, evaluates rules, serves queries (the core) |
| **Retrieval / scrape manager** | Discovers targets and scrapes them at the configured interval |
| **TSDB** | The time-series database on local disk (head block in memory + WAL + persistent blocks) |
| **PromQL engine + HTTP API** | Answers queries from Grafana, UI, scripts |
| **Rules engine** | Evaluates **recording rules** (precompute expensive queries) and **alerting rules** |
| **Exporters** | Programs that expose metrics of systems you can't instrument directly (Linux host, MySQL, Redis, hardware) |
| **Client libraries** | Instrument your own code (Python, Go, Java, Ruby, .NET, Rust...) |
| **Pushgateway** | Intermediate store for **short-lived batch jobs** that push metrics; Prometheus scrapes the gateway |
| **Service discovery (SD)** | Finds targets dynamically (Kubernetes, AWS EC2, Consul, DNS, file-based) |
| **Alertmanager** | Receives alerts from Prometheus; **deduplicates, groups, routes, silences, inhibits** and sends notifications |
| **Grafana** | Visualization (separate project) |
| **Remote storage integrations** | `remote_write`/`remote_read` for long-term storage and global views |

## 2.2 Data flow in one sentence

```
Target exposes /metrics → Prometheus scrapes every N seconds → stores samples in TSDB → rules evaluate → alerts go to Alertmanager → notifications; Grafana queries via PromQL for dashboards.
```

---

# 3️⃣ THE DATA MODEL ⭐⭐⭐

## 3.1 Time series

> A **time series** is a stream of **timestamped values** belonging to the same metric and the same set of labels.

```
metric name + labels  →  (timestamp₁, value₁), (timestamp₂, value₂), (timestamp₃, value₃) ...
```

Notation:

```
<metric_name>{<label_name>="<label_value>", ...}  value  @timestamp

http_requests_total{method="GET", endpoint="/checkout", status="200", instance="10.0.1.5:5000", job="flask-app"}  18240
```

| Piece | Meaning |
|---|---|
| **Metric name** | What is measured (`http_requests_total`); must match `[a-zA-Z_:][a-zA-Z0-9_:]*` (Prometheus 3 also allows UTF-8 names with quoting, but keep conventional names) |
| **Labels** | Key-value pairs that add **dimensions** (method, status, instance...) |
| **Sample** | One data point = a **float64 value** + **millisecond timestamp** |
| **Series identity** | The metric name + the **full set** of labels. Change any label value → a **different** time series |
| **`__name__`** | Internal label holding the metric name (`{__name__="up"}` ≡ `up`) |
| **Labels starting with `__`** | Reserved for internal use (`__address__`, `__meta_*` during relabeling) |

Example: these are **three separate series**:

```
http_requests_total{status="200"}   1200
http_requests_total{status="404"}     35
http_requests_total{status="500"}      7
```

## 3.2 Automatic labels

Every scraped series gets two labels automatically:

| Label | Meaning |
|---|---|
| **`job`** | Logical group of targets with the same purpose (from the `job_name` in config), e.g., `node`, `flask-app` |
| **`instance`** | The specific target `host:port`, e.g., `10.0.1.5:9100` |

## 3.3 Naming conventions ⭐ (good for exams and real life)

```
snake_case                                       http_request_duration_seconds   ✔     httpRequestDuration   ✘
Use BASE UNITS in the name: seconds, bytes, ratio (0-1)   node_memory_MemAvailable_bytes ✔   ..._ms ✘, ..._MB ✘
Counters end with _total                         http_requests_total
Suffix with the unit                              ..._seconds, ..._bytes, ..._ratio
Prefix with the app/subsystem                     myapp_orders_created_total
A metric name describes WHAT is measured; labels describe WHICH/WHERE (don't put label data in the name)
```

## 3.4 Cardinality ⚠️ (the #1 operational danger)

> **Cardinality** = the number of unique time series. Each unique label combination is a separate series stored in memory.

```
http_requests_total with labels: method (5) × endpoint (40) × status (10) × instance (20) = 40,000 series  ✔ fine
Add label user_id (1,000,000 users)  →  40,000 × 1,000,000 = 40 BILLION series  ❌ Prometheus dies
```

Rules:

```
✔ Labels must have a BOUNDED set of values (status code, method, route template, region)
✘ Never use unbounded values as labels: user IDs, emails, session IDs, request IDs, full URLs, timestamps, raw error messages
✔ Put high-cardinality details in LOGS and TRACES, not metrics
✔ Use route templates (/users/{id}) not raw paths (/users/8123)
```

---

# 4️⃣ THE FOUR METRIC TYPES ⭐⭐⭐

## 4.1 Counter 🔢

> A **counter** is a cumulative value that **only goes up** (or resets to zero on restart).

Examples: `http_requests_total`, `errors_total`, `bytes_sent_total`, `node_cpu_seconds_total`.

```
t0: 1000   t1: 1050   t2: 1130   t3: 0 (restart!)   t4: 40
```

You rarely graph a raw counter. You use **`rate()`** (per-second increase), and PromQL handles resets:

```promql
rate(http_requests_total[5m])        # requests per second averaged over 5 minutes
```

## 4.2 Gauge 🌡️

> A **gauge** is a value that can **go up and down**: a current reading.

Examples: `node_memory_MemAvailable_bytes`, `temperature_celsius`, `http_requests_in_progress`, `queue_length`, `node_load1`.

```promql
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes     # use directly, no rate()
```

## 4.3 Histogram 📊

> A **histogram** samples observations (usually durations or sizes) into **configurable buckets** and also tracks a **sum** and a **count**; used to calculate **percentiles** on the server.

One histogram `http_request_duration_seconds` exposes several series:

```
http_request_duration_seconds_bucket{le="0.05"}   240     # observations ≤ 50 ms (cumulative!)
http_request_duration_seconds_bucket{le="0.1"}    410     # ≤ 100 ms
http_request_duration_seconds_bucket{le="0.5"}    498     # ≤ 500 ms
http_request_duration_seconds_bucket{le="+Inf"}   500     # all observations (= _count)
http_request_duration_seconds_sum                 61.3    # total seconds across all observations
http_request_duration_seconds_count               500     # number of observations
```

- Buckets are **cumulative** (`le` = "less than or equal").
- Average latency = `_sum / _count` (via rates).
- Percentile: `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))`.
- Percentiles are **estimated** by interpolation inside the bucket: choose bucket boundaries around your SLO (e.g., a bucket at 0.3 s if SLO is 300 ms).
- **Aggregatable** across instances (big advantage over summaries).
- Newer Prometheus versions also support **native histograms** (sparse, high-resolution, cheaper); classic histograms are still the common default, so learn those first.

## 4.4 Summary 📋

> A **summary** also records observations (`_sum`, `_count`) but calculates **quantiles on the client side** (e.g., `quantile="0.99"`).

```
rpc_duration_seconds{quantile="0.5"}   0.05
rpc_duration_seconds{quantile="0.99"}  0.9
rpc_duration_seconds_sum   ... ; rpc_duration_seconds_count ...
```

Downside: quantiles **cannot be aggregated** across instances (you can't average p99s). Prefer **histograms** unless you need exact client-side quantiles.

## 4.5 Comparison ⭐

| | Counter | Gauge | Histogram | Summary |
|---|---|---|---|---|
| Direction | Only up | Up/down | Buckets (cumulative counters) | Quantiles + sum/count |
| Typical use | Requests, errors, bytes | Memory, temperature, queue size | Latency, sizes | Client-side quantiles |
| Query | `rate()` / `increase()` | Direct / `avg_over_time` | `histogram_quantile(rate(_bucket))` | Read quantile series |
| Aggregatable | Yes (sum of rates) | Yes (sum/avg) | **Yes** | **No** (quantiles) |
| Example | `http_requests_total` | `node_load1` | `http_request_duration_seconds_bucket` | `rpc_duration_seconds` |

## 4.6 Which type to choose? (decision)

```
Counts events that only increase?        → Counter     (name ends with _total)
Current state that goes up and down?     → Gauge
Durations/sizes where you need percentiles across many instances? → Histogram
Need exact quantiles from one process?   → Summary (rare)
```

---

# 5️⃣ THE EXPOSITION FORMAT AND `/metrics` ⭐

Targets expose metrics over **HTTP** at `/metrics` (default) in a simple **text format** (also OpenMetrics, a standardized evolution):

```
# HELP http_requests_total Total HTTP requests.
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/",status="200"} 1027
http_requests_total{method="GET",endpoint="/",status="500"} 3

# HELP node_memory_MemAvailable_bytes Memory information field MemAvailable_bytes.
# TYPE node_memory_MemAvailable_bytes gauge
node_memory_MemAvailable_bytes 1.62947584e+09
```

Check any target by hand:

```bash
curl http://localhost:9100/metrics | head -40          # Node Exporter
curl http://localhost:9090/metrics | grep prometheus_tsdb   # Prometheus's own metrics (it monitors itself)
```

`# HELP` = description, `# TYPE` = metric type.

---

# 6️⃣ JOBS, INSTANCES AND THE `up` METRIC ⭐

| Term | Meaning |
|---|---|
| **Target** | An endpoint Prometheus scrapes (`10.0.1.5:9100`) |
| **Instance** | One target (label `instance`) |
| **Job** | A collection of instances with the same purpose (label `job`), e.g., 5 web servers = job `web` |

For **every scrape**, Prometheus automatically records synthetic series:

| Series | Meaning |
|---|---|
| **`up{job, instance}`** | **1** if the scrape succeeded, **0** if it failed ⭐ (your first availability alert: `up == 0`) |
| `scrape_duration_seconds` | How long the scrape took |
| `scrape_samples_scraped` | Number of samples returned |
| `scrape_samples_post_metric_relabeling` | Samples kept after relabeling |
| `scrape_series_added` | New series added |

```promql
up                         # all targets: 1 = healthy
up == 0                    # targets that are DOWN
count(up == 1) by (job)    # healthy instances per job
```

---

# 7️⃣ CONFIGURATION: `prometheus.yml` ⭐⭐⭐ (PRACTICAL 8 CORE)

## 7.1 Minimal working configuration

```yaml
global:
  scrape_interval: 15s          # how often to scrape targets (default 1m)
  evaluation_interval: 15s      # how often to evaluate rules
  # scrape_timeout: 10s         # must be ≤ scrape_interval
  external_labels:              # attached to alerts and remote-written data (identify this Prometheus)
    monitor: "lab-prometheus"
    env: "dev"

rule_files:
  - "rules/*.yml"               # alerting and recording rules (Part 4)

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["localhost:9093"]

scrape_configs:
  - job_name: "prometheus"      # Prometheus monitors itself
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"            # Node Exporter on this host and another host
    static_configs:
      - targets: ["localhost:9100", "10.0.1.20:9100"]
        labels:
          env: "dev"
```

## 7.2 Section by section

| Section | Purpose |
|---|---|
| **`global`** | Defaults: `scrape_interval`, `scrape_timeout`, `evaluation_interval`, `external_labels` |
| **`rule_files`** | Paths/globs to rule files (recording + alerting rules) |
| **`alerting`** | Where Alertmanager(s) are |
| **`scrape_configs`** | **List of jobs**: what to scrape and how |
| **`remote_write` / `remote_read`** | Send/read data from long-term storage (Part 6) |
| **`storage`** | Advanced TSDB tuning |

Per-job options you'll use: `job_name`, `scrape_interval`, `metrics_path` (default `/metrics`), `scheme` (http/https), `static_configs` (targets + extra labels), service discovery blocks, `basic_auth`, `tls_config`, `relabel_configs`, `metric_relabel_configs`, `honor_labels`, `params` (for exporters such as blackbox).

## 7.3 Choosing intervals

| Interval | Typical | Trade-off |
|---|---|---|
| `scrape_interval` | **15-30 s** (labs: 15 s) | Shorter = finer resolution, more storage/CPU; longer = coarser |
| `evaluation_interval` | same as scrape | Rules run on stored data |
| Rate window `[5m]` | ≥ **4 × scrape_interval** | `rate()` needs enough samples in the window |

## 7.4 Service discovery ⭐ (dynamic cloud!)

Static lists don't survive autoscaling (Unit I): instances come and go. **Service discovery** finds targets automatically.

| Mechanism | Config block | Use |
|---|---|---|
| **Static** | `static_configs` | Labs, fixed servers |
| **File-based** | `file_sd_configs` | JSON/YAML target files updated by scripts/Ansible/Terraform |
| **AWS EC2** | `ec2_sd_configs` | Discover instances by region/tags (needs IAM read permission) |
| **Kubernetes** | `kubernetes_sd_configs` | Pods, services, nodes, endpoints |
| **DNS** | `dns_sd_configs` | SRV/A records |
| **Consul, Azure, GCE, Docker, etc.** | respective blocks | Platform-specific |

EC2 discovery example (tag-based, using relabeling):

```yaml
- job_name: "ec2-nodes"
  ec2_sd_configs:
    - region: ap-south-1
      port: 9100
      filters:
        - name: tag:Monitoring
          values: ["enabled"]
  relabel_configs:
    - source_labels: [__meta_ec2_tag_Name]
      target_label: name
    - source_labels: [__meta_ec2_availability_zone]
      target_label: az
```

File-based discovery (`targets.json`):

```json
[
  { "targets": ["10.0.1.11:9100", "10.0.1.12:9100"], "labels": { "env": "prod", "role": "web" } }
]
```

```yaml
- job_name: "file-nodes"
  file_sd_configs:
    - files: ["/etc/prometheus/targets/*.json"]
      refresh_interval: 1m
```

## 7.5 Relabeling ⭐ (rewrite labels and filter targets)

| Stage | Config | When |
|---|---|---|
| **`relabel_configs`** | **Before** the scrape; on **target** labels (including `__meta_*` from discovery) | Choose which targets to scrape, set `job`/`instance`, add labels |
| **`metric_relabel_configs`** | **After** the scrape; on each **sample's** labels | Drop expensive metrics/labels, rename |

Examples:

```yaml
relabel_configs:
  # keep only targets with a tag Monitoring=enabled
  - source_labels: [__meta_ec2_tag_Monitoring]
    regex: enabled
    action: keep
  # use the Name tag as the instance label
  - source_labels: [__meta_ec2_tag_Name]
    target_label: instance

metric_relabel_configs:
  # drop a noisy high-cardinality metric family
  - source_labels: [__name__]
    regex: "go_gc_.*"
    action: drop
  # drop a label
  - regex: "pod_template_hash"
    action: labeldrop
```

Common actions: `keep`, `drop`, `replace`, `labelmap`, `labeldrop`, `labelkeep`, `hashmod` (sharding, Part 6).

## 7.6 Validate and reload ✅

```bash
promtool check config /etc/prometheus/prometheus.yml        # validate before applying
promtool check rules /etc/prometheus/rules/*.yml            # validate rule files

# reload WITHOUT restart (any one of these):
sudo systemctl reload prometheus                            # if the unit sends SIGHUP
sudo kill -HUP $(pidof prometheus)
curl -X POST http://localhost:9090/-/reload                 # requires --web.enable-lifecycle flag
```

---

# 8️⃣ INSTALLATION OVERVIEW AND KEY FLAGS 💻

Full step-by-step (systemd, users, directories, firewall) is in **Part 8 (lab)**. Summary:

```bash
# Download from https://prometheus.io/download/ (pick the current LTS or latest; version below is an example)
VER=3.13.5          # check the download page; set to the version you choose
wget https://github.com/prometheus/prometheus/releases/download/v${VER}/prometheus-${VER}.linux-amd64.tar.gz
tar xzf prometheus-${VER}.linux-amd64.tar.gz
./prometheus-${VER}.linux-amd64/prometheus --config.file=prometheus.yml
# → UI at http://localhost:9090
```

Docker:

```bash
docker run -d --name prometheus -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```

## 8.1 Important command-line flags ⭐

| Flag | Meaning | Default |
|---|---|---|
| `--config.file` | Config file path | `prometheus.yml` |
| `--storage.tsdb.path` | Data directory | `data/` |
| `--storage.tsdb.retention.time` | How long to keep data | **15d** |
| `--storage.tsdb.retention.size` | Max disk for blocks (e.g., `10GB`); oldest deleted first | none |
| `--web.listen-address` | Bind address:port | `0.0.0.0:9090` (use `127.0.0.1:9090` behind a proxy) |
| `--web.enable-lifecycle` | Enables `/-/reload` and `/-/quit` HTTP endpoints | off |
| `--web.enable-admin-api` | Enables delete-series/snapshot APIs | off |
| `--web.external-url` | Public URL (for links in alerts) | auto |
| `--web.config.file` | TLS/basic-auth config for the web server | none |
| `--log.level` | debug/info/warn/error | info |

---

# 9️⃣ THE WEB UI TOUR 🖥️

Open `http://<server>:9090`:

| Menu | What it shows |
|---|---|
| **Query / Graph** | Type PromQL; **Table** (instant value) or **Graph** (time range); also an Explain/metrics explorer in newer UIs |
| **Alerts** | Alert rules and states: **inactive**, **pending** (condition true, waiting for `for:`), **firing** |
| **Status → Targets** | Every scrape target, state **UP/DOWN**, last scrape time, duration, error message ⭐ (first stop when something is missing) |
| **Status → Service discovery** | Discovered labels before/after relabeling |
| **Status → Configuration** | Loaded config |
| **Status → Rules** | Rule groups, last evaluation, durations |
| **Status → TSDB status** | Top series by metric/label (find **cardinality** culprits) |
| **Status → Runtime & Build information** | Version, retention, uptime |
| **`/metrics`** | Prometheus's own metrics |
| **`/-/healthy`, `/-/ready`** | Health endpoints for load balancers/Kubernetes |

First sanity query: `up` → each target should show **1**.

---

# 🔟 THE TSDB (STORAGE ENGINE) ⭐

```
 incoming samples → HEAD BLOCK (in memory, last ~2 hours) ──also written to──▶ WAL (write-ahead log on disk, crash recovery)
                         │ every 2 hours
                         ▼
                 persistent BLOCKS on disk (each = chunks + index + meta.json) ── compaction merges small blocks into bigger ones ──▶ old blocks deleted by retention
```

| Concept | Meaning |
|---|---|
| **Head block** | Recent data in memory for fast writes/queries |
| **WAL** | Write-ahead log: replayed after a crash/restart so no data lost (restart can take time with large heads) |
| **Blocks** | Immutable 2-hour (and larger after compaction) directories (`01H...`) |
| **Compaction** | Merges blocks, saves space, speeds up queries |
| **Retention** | Time (`15d` default) and/or size limit; old blocks removed |
| **Compression** | ~1-2 bytes per sample on average (Gorilla-style compression) |

## 10.1 Disk sizing formula ⭐

```
needed_disk_space = retention_time_seconds × ingested_samples_per_second × bytes_per_sample (≈1-2)

Example: 15 days, 100,000 series scraped every 15 s
   samples/second = 100,000 / 15 ≈ 6,667
   15 days = 1,296,000 s
   disk ≈ 1,296,000 × 6,667 × 2 bytes ≈ 17.3 GB
```

Memory is driven mostly by the **number of active series** (rule of thumb: a few KB per series in the head). Watch `prometheus_tsdb_head_series`.

## 10.2 Operational notes

```
Local storage is NOT clustered/replicated: use remote_write + Thanos/Mimir for durability (Part 6); keep EBS snapshots/backups for labs
Don't put the data directory on network file systems (NFS); use local SSD/EBS
Default retention 15d: raise it deliberately and size the disk
```

---

# 1️⃣1️⃣ EXPORTERS ⭐⭐⭐

> An **exporter** is a small program that **collects metrics from a system that doesn't expose Prometheus metrics itself**, and serves them at `/metrics`.

## 11.1 Common exporters

| Exporter | Monitors | Default port |
|---|---|---|
| **node_exporter** | Linux host: CPU, memory, disk, network, load, filesystems | **9100** |
| **windows_exporter** | Windows hosts | 9182 |
| **blackbox_exporter** | Probes endpoints from outside (HTTP/HTTPS, TCP, ICMP, DNS) | 9115 |
| **cAdvisor** | Container resource usage (Docker/Kubernetes) | 8080 |
| **kube-state-metrics** | Kubernetes object states (deployments, pods) | 8080 |
| **mysqld_exporter, postgres_exporter, mongodb_exporter, redis_exporter** | Databases/caches | 9104 / 9187 / 9216 / 9121 |
| **nginx-prometheus-exporter**, **apache_exporter** | Web servers | 9113 / 9117 |
| **jmx_exporter** | JVM apps | custom |
| **snmp_exporter** | Network devices | 9116 |
| **cloudwatch_exporter / yet-another-cloudwatch-exporter** | Pull AWS CloudWatch metrics into Prometheus | 9106 |
| **Jenkins Prometheus plugin** | Jenkins metrics at `/prometheus` (Unit III link) | via Jenkins |

Rule: before writing code, **search for an existing exporter**.

## 11.2 Node Exporter in depth (Practical 8) 🖥️

Exposes hundreds of host metrics via **collectors** (enabled by default: cpu, meminfo, diskstats, filesystem, netdev, loadavg, uname, time, etc.).

| What you want | Key metrics (names) | Type | Notes |
|---|---|---|---|
| **CPU** | `node_cpu_seconds_total{cpu, mode}` (modes: idle, user, system, iowait, irq, softirq, steal, nice) | counter | CPU % = 100 − idle% (Part 3) |
| **Load** | `node_load1`, `node_load5`, `node_load15` | gauge | Compare with number of CPUs |
| **Memory** | `node_memory_MemTotal_bytes`, `node_memory_MemAvailable_bytes`, `node_memory_MemFree_bytes`, `node_memory_Buffers_bytes`, `node_memory_Cached_bytes`, `node_memory_SwapTotal_bytes`, `node_memory_SwapFree_bytes` | gauge | Use **MemAvailable** (not Free) for "usable" memory |
| **Disk space** | `node_filesystem_size_bytes`, `node_filesystem_avail_bytes`, `node_filesystem_free_bytes`, `node_filesystem_files_free` (labels: `mountpoint`, `fstype`, `device`) | gauge | Exclude tmpfs/overlay noise with `fstype!~"tmpfs|overlay"` |
| **Disk I/O** | `node_disk_read_bytes_total`, `node_disk_written_bytes_total`, `node_disk_io_time_seconds_total`, `node_disk_reads_completed_total` | counter | I/O utilization = `rate(node_disk_io_time_seconds_total[5m])` |
| **Network** | `node_network_receive_bytes_total{device}`, `node_network_transmit_bytes_total`, `node_network_receive_errs_total`, `..._drop_total` | counter | Exclude `lo` |
| **System info** | `node_uname_info`, `node_boot_time_seconds`, `node_time_seconds`, `node_filefd_allocated` | gauge | Uptime = `time() - node_boot_time_seconds` |
| **Pressure** | `node_pressure_cpu_waiting_seconds_total`, memory/io pressure (PSI) | counter | Saturation signals on newer kernels |

Useful runtime flags:

```bash
node_exporter --web.listen-address=":9100" \
              --collector.textfile.directory=/var/lib/node_exporter/textfile \
              --no-collector.wifi \
              --collector.systemd         # opt-in: systemd unit states
node_exporter --help | grep collector      # list collectors
```

## 11.3 Textfile collector (custom metrics from scripts) ⭐

Cron job writes `.prom` files; Node Exporter exposes them. Great for backups, batch jobs, certificates:

```bash
# /usr/local/bin/backup_metric.sh  (run by cron after the backup job)
echo "backup_last_success_timestamp_seconds $(date +%s)" > /var/lib/node_exporter/textfile/backup.prom.$$
mv /var/lib/node_exporter/textfile/backup.prom.$$ /var/lib/node_exporter/textfile/backup.prom   # atomic write
```

Alert later if `time() - backup_last_success_timestamp_seconds > 86400`.

## 11.4 Blackbox exporter (black-box probes) 🕵️

```yaml
# prometheus.yml
- job_name: "blackbox-http"
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets: ["https://example.com", "http://localhost:80"]
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: localhost:9115          # the blackbox exporter address
```

Gives `probe_success`, `probe_duration_seconds`, `probe_http_status_code`, `probe_ssl_earliest_cert_expiry` (alert before certificates expire).

---

# 1️⃣2️⃣ INSTRUMENTING YOUR OWN APP (PYTHON/FLASK) ⭐⭐⭐

White-box monitoring means **your code emits metrics**. Install:

```bash
pip install flask prometheus-client
```

`app.py` (extends the Unit III Flask app; this is also the target for dashboards in Parts 5 and 8):

```python
import random
import time

from flask import Flask, Response, jsonify, request
from prometheus_client import (CONTENT_TYPE_LATEST, Counter, Gauge, Histogram,
                               generate_latest)

app = Flask(__name__)

# ---------- metric definitions (create ONCE at module level) ----------
REQUESTS = Counter(
    "myapp_http_requests_total", "Total HTTP requests",
    ["method", "endpoint", "status"],
)
LATENCY = Histogram(
    "myapp_http_request_duration_seconds", "Request latency in seconds",
    ["endpoint"],
    buckets=(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0),
)
IN_PROGRESS = Gauge("myapp_http_requests_in_progress", "Requests currently being processed")
ORDERS = Counter("myapp_orders_created_total", "Orders created (business metric)")


# ---------- automatic instrumentation for every request ----------
@app.before_request
def before():
    request.start_time = time.perf_counter()
    IN_PROGRESS.inc()


@app.after_request
def after(response):
    if request.path != "/metrics":
        # use the ROUTE TEMPLATE (low cardinality), not the raw URL
        endpoint = request.url_rule.rule if request.url_rule else "unmatched"
        elapsed = time.perf_counter() - request.start_time
        LATENCY.labels(endpoint).observe(elapsed)
        REQUESTS.labels(request.method, endpoint, str(response.status_code)).inc()
    IN_PROGRESS.dec()
    return response


# ---------- application routes ----------
@app.route("/")
def home():
    return "<h1>Instrumented Flask app</h1><p>See /metrics</p>"


@app.route("/health")
def health():
    return jsonify(status="ok")


@app.route("/order", methods=["POST", "GET"])
def order():
    time.sleep(random.uniform(0.01, 0.15))      # simulate work
    ORDERS.inc()
    return jsonify(status="created")


@app.route("/slow")
def slow():
    time.sleep(random.uniform(0.3, 1.5))        # simulate a slow endpoint
    return jsonify(status="slow but ok")


@app.route("/flaky")
def flaky():
    if random.random() < 0.3:                   # 30% of requests fail
        return jsonify(error="boom"), 500
    return jsonify(status="ok")


# ---------- the scrape endpoint ----------
@app.route("/metrics")
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Run and generate traffic:

```bash
python app.py &
for i in $(seq 1 50); do curl -s localhost:5000/order > /dev/null; curl -s localhost:5000/flaky > /dev/null; curl -s localhost:5000/slow > /dev/null; done
curl -s localhost:5000/metrics | grep myapp_ | head -30
```

Scrape it:

```yaml
- job_name: "flask-app"
  static_configs:
    - targets: ["localhost:5000"]
```

Then in the Prometheus UI (Part 3 explains each):

```promql
sum(rate(myapp_http_requests_total[1m])) by (endpoint)                        # request rate per endpoint
sum(rate(myapp_http_requests_total{status=~"5.."}[1m])) / sum(rate(myapp_http_requests_total[1m]))   # error ratio
histogram_quantile(0.95, sum by (le, endpoint) (rate(myapp_http_request_duration_seconds_bucket[1m])))   # p95 latency
```

### Notes for real deployments

```
• gunicorn with multiple worker processes needs prometheus_client multiprocess mode (PROMETHEUS_MULTIPROC_DIR) or a per-worker exporter strategy
• Library alternative: prometheus-flask-exporter gives these RED metrics with 2 lines of code
• Other languages: Go (client_golang), Java (Micrometer/Prometheus client), Node.js (prom-client), .NET (prometheus-net)
• OpenTelemetry SDKs can also export Prometheus-format metrics (Part 7)
• Don't create metrics inside request handlers (duplicate registration errors); define once at import time
```

### Instrumentation best practices ✅

```
Instrument the 4 golden signals first (RED) → then business metrics → then internals
Bounded labels only ; route templates not raw paths ; base units (seconds, bytes) ; counters end with _total
Histogram buckets aligned with SLO thresholds ; expose /health separately from /metrics
Make metrics cheap (no heavy computation on scrape) ; protect /metrics from the public internet
```

---

# 1️⃣3️⃣ PUSHGATEWAY (SHORT-LIVED JOBS) ⚠️

For **batch jobs that finish before Prometheus can scrape them** (nightly backup, CI job), the job **pushes** its final metrics to the Pushgateway, and Prometheus scrapes the gateway.

```bash
# a batch job pushing its result
cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/nightly_backup/instance/db1
# TYPE backup_duration_seconds gauge
backup_duration_seconds 312
# TYPE backup_last_success_timestamp_seconds gauge
backup_last_success_timestamp_seconds $(date +%s)
EOF
```

```yaml
- job_name: "pushgateway"
  honor_labels: true          # keep the job/instance labels set by the pusher
  static_configs:
    - targets: ["localhost:9091"]
```

| Use it for | Don't use it for |
|---|---|
| Service-level batch jobs (cron/CI) whose result you need | Converting Prometheus to a push system for normal services |
| | Per-instance metrics of long-running services (the gateway never forgets: stale series remain, and it becomes a single point of failure; delete old groups) |

Alternative for host-level batch jobs: **Node Exporter textfile collector**.

---

# 1️⃣4️⃣ SECURITY BASICS 🔐 (BRIDGE TO UNIT V)

```
Prometheus and exporters have NO authentication by default and expose sensitive info (hostnames, topology, versions).
✔ Bind to localhost or a private interface; restrict Security Groups (9090, 9100, 9093 only from trusted IPs/VPC)
✔ Put a reverse proxy (Nginx) with TLS + auth in front, or use --web.config.file (TLS + basic auth, bcrypt hashes)
✔ Scrape over TLS/authenticated endpoints when crossing networks (tls_config, basic_auth in scrape config)
✔ Don't expose /metrics of apps publicly; don't put secrets/PII in metric labels
✔ Disable --web.enable-admin-api and --web.enable-lifecycle unless needed and protected
✔ Keep Prometheus and exporters updated (security advisories exist)
```

`web.yml` example (basic auth):

```yaml
basic_auth_users:
  admin: $2y$10$...bcrypt-hash...      # generate with: htpasswd -nBC 10 "" | tr -d ':\n'
```

---

# 1️⃣5️⃣ LIMITATIONS AND TRADE-OFFS ⚖️

| Limitation | Workaround |
|---|---|
| Single-node local storage, no clustering | HA pairs + remote write + Thanos/Mimir/VictoriaMetrics (Part 6) |
| Limited long-term retention | Remote storage, downsampling |
| Pull model needs network reachability | Pushgateway (batch), agents/remote write, OTel Collector |
| Not for logs/traces | Loki/Tempo (Part 7) |
| Rates/percentiles are approximations | Understand bucket choices; use SLO-aligned buckets |
| High-cardinality risk | Label hygiene, relabeling drops, TSDB status checks |
| Scrape gaps if Prometheus is down | Run two replicas |

---

# 1️⃣6️⃣ DRY RUN: ONE SCRAPE CYCLE 🏃

```
t=00s  Prometheus (scrape_interval 15s) knows target 10.0.1.5:9100 for job "node" (static config)
t=15s  Scrape manager sends:  GET http://10.0.1.5:9100/metrics   (timeout 10s)
       Node Exporter reads /proc and /sys, returns ~1,000 lines of text
       Prometheus parses each line → (series, timestamp=now, value)
       Adds labels job="node", instance="10.0.1.5:9100"  (+ relabeling rules, if any)
       Records up{job="node",instance="10.0.1.5:9100"} = 1 and scrape_duration_seconds=0.021
       Appends samples to the HEAD block (and WAL)
t=15s  Rules engine (every 15 s) evaluates: node_filesystem_avail_bytes ... < 10% ?  → pending/firing → sends alerts to Alertmanager
t=20s  Grafana asks via HTTP API: rate(node_cpu_seconds_total{mode="idle"}[5m]) → PromQL engine reads head+blocks → returns series → panel renders
t=2h   Head block compacted to a persistent block on disk; WAL truncated
t=15d  Retention deletes the oldest block
If the target is down at t=30s: scrape fails → up = 0 → alert "InstanceDown" after the "for:" duration
```

---

# 1️⃣7️⃣ COMMON MISTAKES 🚨

| Mistake | Effect | Fix |
|---|---|---|
| Wrong `scrape_timeout` (> interval) | Config error | Timeout ≤ interval |
| Target shows DOWN: `connection refused` | Exporter not running / wrong port / firewall | `systemctl status`, `curl :9100/metrics`, open Security Group between hosts |
| Target DOWN: `context deadline exceeded` | Slow target/timeout/network | Increase timeout, check load/network |
| YAML indentation/tab errors | Prometheus won't start | `promtool check config` |
| Changed config but nothing happened | Not reloaded | Reload (`/-/reload`, SIGHUP, restart) |
| `rate()` on a gauge / using raw counters | Meaningless graphs | Counters → `rate()`; gauges direct |
| `rate()` window too small (`[30s]` with 15 s scrape) | Empty/jumpy result | Window ≥ 4× scrape interval |
| High-cardinality labels | Memory explosion | Bounded labels; drop via `metric_relabel_configs` |
| Prometheus data dir on tiny root disk | Disk full, crash | Separate volume; retention/size limits |
| Port 9090/9100 open to the world | Information leak | Restrict network, auth/TLS |
| Instrumenting in request handler → duplicate timeseries error | App crash | Define metrics once at module level |
| Using `localhost` targets from inside Docker | Wrong network namespace | Use container names/host IP/`host.docker.internal` |

---

# 1️⃣8️⃣ VIVA QUESTIONS (PART 2) 🎯

**Q1. What is Prometheus?** An open-source, pull-based monitoring and alerting system with a multi-dimensional time-series database and PromQL, a CNCF graduated project.

**Q2. Explain Prometheus architecture.** Server (scrape/retrieval, TSDB, PromQL, rules engine) pulls metrics from targets/exporters, uses service discovery, sends alerts to Alertmanager, serves queries to Grafana; Pushgateway handles short-lived jobs; remote write for long-term storage.

**Q3. What is a time series and how is it identified?** A stream of timestamped values identified by metric name plus the full set of label key-value pairs.

**Q4. Name the four metric types.** Counter, gauge, histogram, summary.

**Q5. Counter vs gauge?** Counter only increases (use `rate()`); gauge goes up and down (use directly).

**Q6. Histogram vs summary?** Histogram uses server-side quantile estimation from cumulative buckets and is aggregatable; summary computes client-side quantiles that can't be aggregated.

**Q7. What do `job` and `instance` labels mean?** Job is a group of targets with the same purpose; instance is a single target `host:port`.

**Q8. What does the `up` metric show?** 1 if the last scrape succeeded, 0 if it failed.

**Q9. What is an exporter? Give examples.** A program exposing metrics for systems that can't be instrumented directly: node_exporter, blackbox_exporter, mysqld_exporter, cAdvisor.

**Q10. What is cardinality and why dangerous?** The number of unique time series; unbounded label values multiply series and exhaust memory.

**Q11. What are `relabel_configs` vs `metric_relabel_configs`?** The first rewrites/filters targets before scraping; the second rewrites/drops samples after scraping.

**Q12. Default retention and how to change it?** 15 days; `--storage.tsdb.retention.time` and/or `--storage.tsdb.retention.size`.

**Q13. Push vs pull, and when to use Pushgateway?** Prometheus pulls; Pushgateway is only for short-lived batch jobs that can't be scraped.

**Q14. How do you reload the configuration?** SIGHUP, or `POST /-/reload` with `--web.enable-lifecycle`, after `promtool check config`.

**Q15. How would you monitor your own application?** Use a client library to expose a `/metrics` endpoint with counters, gauges and histograms for RED metrics and business events, then add it as a scrape job.

**Q16. How do you size Prometheus disk?** Retention seconds × samples per second × ~1-2 bytes per sample.

---

# 🧠 PART 2 MASTER SHEET

```
PROMETHEUS: open-source, pull-based, TSDB + PromQL + Alertmanager ; SoundCloud 2012 ; CNCF graduated ; Go binary ; 3.x line with LTS releases
ARCHITECTURE: scrape manager + service discovery → TSDB (head + WAL + 2h blocks) → PromQL/HTTP API (Grafana) ; rules → Alertmanager ; Pushgateway ; remote write
DATA MODEL: metric_name{label="v"} value @ms-timestamp ; series = name + ALL labels ; auto labels job, instance ; __name__
NAMING: snake_case, base units (_seconds, _bytes), counters _total ; CARDINALITY: bounded labels only (no user_id)
TYPES: Counter (up only → rate) | Gauge (up/down) | Histogram (_bucket{le}, _sum, _count → histogram_quantile, aggregatable) | Summary (client quantiles, not aggregatable)
/metrics text format (# HELP, # TYPE) ; up (1/0) ; scrape_duration_seconds
CONFIG: global (scrape_interval 15s, evaluation_interval, external_labels) · rule_files · alerting · scrape_configs (static, file_sd, ec2_sd, kubernetes_sd)
RELABEL: relabel_configs (targets, pre-scrape) vs metric_relabel_configs (samples, post-scrape) ; actions keep/drop/replace/labelmap/labeldrop
CLI: promtool check config/rules ; reload via SIGHUP or /-/reload (--web.enable-lifecycle) ; retention 15d default ; disk = retention × samples/s × ~2B
EXPORTERS: node_exporter :9100 (node_cpu_seconds_total, node_memory_MemAvailable_bytes, node_filesystem_avail_bytes, node_network_*_bytes_total, node_load1) ;
           blackbox :9115 ; cAdvisor ; DB exporters ; textfile collector ; Pushgateway :9091 (batch only)
INSTRUMENT: prometheus_client (Counter/Gauge/Histogram), /metrics route, route templates, define once
SECURITY: no auth by default → private network, TLS/basic auth, restrict ports
```

---

## 🫡 PART 2 STATUS

You understand Prometheus architecture, the data model and metric types, configuration and service discovery, the TSDB, exporters (especially Node Exporter), and how to instrument your own Flask app.

**STOP here.** Next up:

> **PART 3 — PromQL Deep Dive (selectors, rate/increase, aggregation, histograms, recording rules, node-exporter queries) 🧮**
