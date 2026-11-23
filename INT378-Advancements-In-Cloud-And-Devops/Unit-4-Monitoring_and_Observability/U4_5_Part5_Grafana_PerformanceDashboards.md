# 📊 UNIT IV — PART 5: GRAFANA AND PERFORMANCE DASHBOARDS

> **Covered in this document:** what Grafana is (origin, licence, editions, the LGTM stack) → architecture and file layout → installation (APT, Docker, binary) and first login → UI tour → **data sources** (connecting Prometheus, other sources) → dashboards, panels and rows → **panel types** and when to use each → the **query editor** (builder vs code, `$__rate_interval`, legends, instant vs range) → **transformations** → **field configuration** (units, thresholds, mappings, overrides, colors) → **variables/templating** (`label_values`, multi-select, repeat) → **annotations** (deployments, alerts, Jenkins integration) → **building the full Infrastructure Performance dashboard** panel by panel for Practical 9 (overview, CPU, memory, storage, network) → importing community dashboards (Node Exporter Full) → a Flask RED dashboard → **dashboard design principles** (audience, hierarchy, USE/RED layout, the 5-second test, anti-patterns) → **provisioning as code** (data sources, dashboards, Git, Terraform) → **Grafana Alerting** (rules, contact points, notification policies) vs Prometheus alerting → users, teams, permissions and security hardening → a Docker Compose lab stack → troubleshooting table → viva → master sheet.

---

Absolutely mate. 😤🔥 Prometheus stores and queries data; **Grafana makes it visible**. This part covers the "creation of performance dashboards" in your syllabus and is the core of **Practical 9**.

---

# 1️⃣ WHAT IS GRAFANA? ⭐

> **Grafana** is an open-source **visualization and observability platform** that connects to many **data sources** (Prometheus, Loki, Tempo, CloudWatch, Elasticsearch, SQL databases...) and lets you build **dashboards**, explore data, and manage **alerts**.

Key facts:

```
Created by Torkel Ödegaard (2014) ; maintained by Grafana Labs ; written in Go (backend) + React/TypeScript (frontend)
Licence: AGPLv3 (since 2021) for the open-source code ; Grafana Cloud = hosted SaaS
Editions: Grafana OSS and Grafana Enterprise (Enterprise is now the default/recommended download; it is free to use and
          functionally identical to OSS unless you add a paid licence for Enterprise-only features)
Default port: 3000      Default login: admin / admin  (you are forced to change it on first login)
Does NOT store metrics itself: it QUERIES data sources (it stores dashboards, users and settings in its own small database)
```

## 1.1 Prometheus vs Grafana (don't mix them up)

| | Prometheus | Grafana |
|---|---|---|
| Job | **Collect, store, query, alert** on metrics | **Visualize** (and explore/alert) data from many sources |
| Data | Owns the time-series database | Owns dashboards/users, **queries** other systems |
| Query language | PromQL | Uses each source's language (PromQL, LogQL, SQL...) |
| UI | Basic expression browser | Rich dashboards, panels, variables |
| Port | 9090 | 3000 |

## 1.2 The Grafana "LGTM" stack (Part 7 preview)

```
L = Loki (logs)   G = Grafana (visualization)   T = Tempo (traces)   M = Mimir (scalable metrics, Prometheus-compatible)
+ Pyroscope (profiles), Alloy (the collector that replaced Grafana Agent), Faro (frontend/RUM), OnCall/IRM (incident response; check current status)
```

---

# 2️⃣ ARCHITECTURE AND FILE LAYOUT 🏗️

```
Browser ──HTTP :3000──▶ GRAFANA SERVER ──┬── queries ──▶ Prometheus (PromQL)
                          │               ├── queries ──▶ Loki / Tempo / CloudWatch / SQL / Elasticsearch ...
                          │               └── alert evaluation ──▶ contact points (email/Slack/webhook)
                          ▼
                 Grafana DB (SQLite by default; MySQL/PostgreSQL for HA) — dashboards, users, orgs, data source definitions
```

| Component | Role |
|---|---|
| **Grafana server** | Web UI + HTTP API + backend query proxy + alert evaluation |
| **Data source plugins** | Adapters for each backend; the Prometheus plugin ships built in |
| **Database** | SQLite by default (`/var/lib/grafana/grafana.db`); use PostgreSQL/MySQL for multi-instance HA |
| **Provisioning** | YAML/JSON files that create data sources/dashboards/alerting at startup (config as code) |
| **Orgs, teams, users, roles** | Multi-tenancy and access control |
| **Plugins** | Panels, data sources, apps (installed with `grafana cli plugins install`) |

## 2.1 Important paths (Linux package install)

| Path | Purpose |
|---|---|
| `/etc/grafana/grafana.ini` | Main configuration |
| `/etc/grafana/provisioning/{datasources,dashboards,alerting,plugins}/` | Provisioning files |
| `/var/lib/grafana/` | Data (SQLite DB, plugins) |
| `/var/log/grafana/grafana.log` | Logs |
| `/usr/share/grafana/` | Binaries, public assets |
| Service | `grafana-server` (systemd) |

Config can also be set with **environment variables** `GF_<SECTION>_<KEY>` (e.g., `GF_SECURITY_ADMIN_PASSWORD`, `GF_SERVER_HTTP_PORT`), which is how containers are configured.

---

# 3️⃣ INSTALLATION 💻

(Full lab steps with verification are in **Part 8**. Always confirm current commands at grafana.com/docs → "Install Grafana on Debian or Ubuntu".)

## 3.1 APT (Ubuntu/Debian) ⭐

```bash
sudo apt-get install -y apt-transport-https wget gnupg
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc
echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana              # OSS package ("grafana-enterprise" is the recommended default edition per Grafana docs)
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```

(Grafana changed its signing-key file name over time, e.g. `gpg.key` vs `gpg-full.key`. If `apt update` shows a signature error, copy the exact key URL from the docs page.)

## 3.2 Docker

```bash
docker run -d --name grafana -p 3000:3000 -v grafana-data:/var/lib/grafana grafana/grafana
# (grafana/grafana-enterprise is the other image name)
```

## 3.3 First login

```
Browser: http://<server-ip>:3000 → username: admin   password: admin → set a NEW strong password
```

⚠️ Open port **3000** only to your IP (Security Group), and move behind HTTPS for anything real.

---

# 4️⃣ UI TOUR 🧭

| Menu | What it does |
|---|---|
| **Home** | Starting page, recent/starred dashboards |
| **Dashboards** | Browse, search, folders, import, playlists, library panels |
| **Explore** | Ad-hoc querying without saving a dashboard (great for PromQL experiments, logs, traces) |
| **Alerting** | Alert rules, contact points, notification policies, silences, mute timings |
| **Connections → Data sources** | Add/test data sources |
| **Administration** | Users, teams, service accounts, plugins, settings, orgs |
| **Profile** | Preferences (theme, timezone), API keys/tokens |

Keyboard help: press `?`. Dark/light theme: profile preferences.

---

# 5️⃣ DATA SOURCES: CONNECTING PROMETHEUS ⭐⭐⭐ (PRACTICAL 9 CORE)

> A **data source** tells Grafana **where** to fetch data and **how** to authenticate.

## 5.1 Add Prometheus (UI)

```
Connections → Data sources → Add data source → Prometheus
  Name:                 Prometheus
  Prometheus server URL: http://localhost:9090          (same server)
                         http://<PROMETHEUS_PRIVATE_IP>:9090   (different server)
                         http://prometheus:9090         (Docker Compose service name)
  Authentication:        none (lab) / basic auth / TLS as configured
  Scrape interval:       15s        ← MUST match Prometheus scrape_interval (affects $__interval and rate windows)
  HTTP method:           POST (default for new versions) or GET
  Default?               ✔ (make it the default data source)
→ Save & test  →  "Successfully queried the Prometheus API."  ✅
```

Important: with the default **Server (proxy)** access, requests go **from the Grafana server to Prometheus**, so `localhost` means the Grafana server's own machine (not your browser).

## 5.2 Common data sources

| Data source | Used for |
|---|---|
| **Prometheus** (and Mimir, Thanos, VictoriaMetrics, AMP via Prometheus-compatible API) | Metrics (PromQL) |
| **Loki** | Logs (LogQL) |
| **Tempo / Jaeger / Zipkin** | Traces |
| **CloudWatch / Azure Monitor / Google Cloud Monitoring** | Cloud-native metrics and logs (Units I and VI) |
| **Elasticsearch / OpenSearch** | Logs and documents |
| **MySQL / PostgreSQL / MSSQL** | Business data via SQL |
| **InfluxDB, Graphite, OpenTSDB** | Other time-series stores |
| **TestData** | Fake data for learning |

## 5.3 Mixed data source

A single panel can combine queries from several sources (`-- Mixed --`): e.g., Prometheus CPU line + a SQL deployment table.

---

# 6️⃣ DASHBOARDS, ROWS AND PANELS 🧱

```
DASHBOARD  (a page: title, tags, folder, time range, refresh, variables, annotations)
  ├── ROW "Overview"      (collapsible group)
  │     ├── PANEL: Stat "Nodes up"        ├── PANEL: Gauge "CPU now"      ├── PANEL: Stat "Uptime"
  ├── ROW "CPU"
  │     ├── PANEL: Time series "CPU usage %"     ├── PANEL: Time series "Load"
  └── ROW "Memory" ...
```

| Concept | Meaning |
|---|---|
| **Panel** | One visualization with one or more queries |
| **Row** | Collapsible section (also used for repeating by variable) |
| **Folder** | Organizes dashboards and carries permissions |
| **Time picker + refresh** | Top-right: "Last 6 hours", auto-refresh (e.g., 30s); keep refresh ≥ scrape interval |
| **JSON model** | Every dashboard is JSON (`Dashboard settings → JSON Model`): copy, version, provision |
| **Versions** | History with diff and restore |
| **Tags, links, library panels, playlists, snapshots, public/shared dashboards** | Organization and sharing |

---

# 7️⃣ PANEL TYPES: WHAT TO USE WHEN ⭐⭐

| Panel | Best for | Example |
|---|---|---|
| **Time series** | Trends over time (the workhorse) | CPU %, request rate, latency |
| **Stat** | One big number (+ optional sparkline) | "Nodes up: 3", "Uptime: 12 d" |
| **Gauge / Bar gauge** | Current value against thresholds | Memory used %, disk used per mount |
| **Table** | Exact values, many series side by side | Top disks by usage, per-instance summary |
| **Bar chart** | Compare categories | Requests per endpoint |
| **Pie/Donut** | Parts of a whole (use sparingly, ≤ 5 slices) | Traffic by status class |
| **Heatmap** | Distribution over time (histograms) | Request latency buckets |
| **Histogram** | Distribution of values | Latency distribution |
| **State timeline / Status history** | Discrete states over time | up/down per target, deployment states |
| **Logs** | Log lines (Loki/Elasticsearch) | Error logs next to metrics |
| **Alert list** | Currently firing alerts | NOC wallboard |
| **Text** | Markdown instructions, runbook links | Dashboard guide |
| **Geomap, Node graph, Canvas, Flame graph, Traces** | Geography, topology, custom layouts, profiles, spans | Service map, traces |

Rule of thumb: **Stat/Gauge for "now"**, **Time series for "how did it change"**, **Table for "exact details"**.

---

# 8️⃣ THE QUERY EDITOR ⭐⭐

For a Prometheus panel you choose **Builder** (point and click) or **Code** (type PromQL; recommended once you know Part 3).

| Option | Meaning |
|---|---|
| **Metric / Label filters / Operations** (builder) | Visual query building |
| **Code mode** | Type PromQL with autocompletion |
| **Legend** | `{{instance}}` or `{{mode}} on {{instance}}` → readable series names |
| **Format** | **Time series** (graphs), **Table** (tables), **Heatmap** (histogram buckets) |
| **Type** | **Range** (graphs), **Instant** (single value for Stat/Table), **Both** |
| **Min step / Resolution / Max data points** | Control query resolution and cost |
| **Exemplars** | Show linked trace IDs on a metric (Part 7) |

## 8.1 Grafana's built-in variables for queries ⭐

| Variable | Meaning |
|---|---|
| **`$__rate_interval`** | The recommended **range for `rate()`**: automatically ≥ 4× scrape interval and adapts to zoom; use `rate(x[$__rate_interval])` |
| `$__interval` | Time between points of the graph (based on dashboard width and range) |
| `$__range` | The dashboard's selected time range (e.g., `increase(x[$__range])`) |
| `$__from`, `$__to` | Range boundaries (ms) |
| `$instance`, `$job` ... | Your own dashboard variables |

Example:

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle", instance=~"$instance"}[$__rate_interval])) * 100)
```

⚠️ If you use a fixed `[5m]` and the dashboard shows a very short range, results may look flat; `$__rate_interval` avoids that. Ensure the data source's **Scrape interval** field matches Prometheus.

---

# 9️⃣ TRANSFORMATIONS 🔧

> **Transformations** reshape query results **inside Grafana** (no query change needed): useful for tables and combining queries.

| Transformation | Use |
|---|---|
| **Organize fields** | Rename, reorder, hide columns |
| **Merge / Join by field** | Combine several queries into one table by `instance` |
| **Reduce** | Convert series to rows: Last, Mean, Max, Min per series |
| **Filter by value / Filter data by query** | Hide rows below a threshold |
| **Group by / Sort by / Limit** | Top-N tables |
| **Add field from calculation** | Create a column, e.g., used = total − free |
| **Labels to fields / Rename by regex** | Turn Prometheus labels into columns, clean names |
| **Convert field type** | Strings ↔ numbers/time |

Typical **"per-host summary table"**: three instant queries (CPU %, memory %, disk %) with Format = Table → transformation **Merge** → **Organize fields** (keep `instance`, `Value #A`, `Value #B`, `Value #C`; rename to CPU %, Memory %, Disk %) → add **thresholds/cell colors** in field config.

---

# 🔟 FIELD CONFIGURATION: UNITS, THRESHOLDS, MAPPINGS, OVERRIDES ⭐⭐

Right panel options (**Standard options**):

| Setting | What to choose | Why |
|---|---|---|
| **Unit** | `Percent (0-100)` when query returns 0-100; `Percent (0.0-1.0)` when it returns a ratio; `bytes (IEC)`; `bits/sec (SI)`; `seconds (s)`; `requests/sec (rps)`; `short` | Correct, readable axis and tooltips (wrong unit is the #1 beginner mistake) |
| **Min / Max** | 0 / 100 for percentages | Stable scale; gauges need it |
| **Decimals** | 1 or 2 | Clean numbers |
| **Thresholds** | Base **green**; **70 → yellow**; **90 → red** (adjust per metric) | Color-coded severity (also draw threshold lines on graphs: "Show thresholds → as lines") |
| **Value mappings** | `1 → "UP" (green)`, `0 → "DOWN" (red)` | Make `up` readable |
| **Color scheme** | From thresholds / classic palette | Consistency |
| **Overrides** | Per-series/field exceptions (e.g., make `iowait` red, right-axis for a second unit) | Fine control |
| **Graph styles** (time series) | Lines/bars, fill opacity, line width, **stack series** (for memory/CPU modes), point size | Readability |
| **Legend** | Table mode with Last/Mean/Max calcs; right/bottom placement | Compact summaries |
| **Tooltip** | All series (shared) | Compare at a point in time |

---

# 1️⃣1️⃣ VARIABLES (TEMPLATING) ⭐⭐⭐

> **Variables** are dropdowns at the top of a dashboard; panels use them in queries. One dashboard then serves **all hosts/jobs/environments**.

## 11.1 Create a variable

```
Dashboard settings → Variables → Add variable
  Type:  Query
  Name:  instance                       (used as $instance)
  Label: Host
  Data source: Prometheus
  Query: label_values(node_uname_info, instance)         ← classic; or label_values(up{job="node"}, instance)
  Refresh: On dashboard load (or On time range change)
  Selection options: ☑ Multi-value   ☑ Include All option
```

Other useful variables:

| Variable | Query/value |
|---|---|
| `$job` | `label_values(up, job)` |
| `$env` | `label_values(node_uname_info, env)` (needs your `env` label) |
| `$mountpoint` | `label_values(node_filesystem_avail_bytes{instance=~"$instance"}, mountpoint)` (chained: depends on `$instance`) |
| `$interval` | Type **Interval**: `1m,5m,15m` |
| `$datasource` | Type **Data source** → pick Prometheus instances (dev/prod) |
| Custom | Fixed list: `critical,warning` |

## 11.2 Use in queries

```promql
node_memory_MemAvailable_bytes{instance=~"$instance"}           # =~ works for multi-value and All
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle", instance=~"$instance"}[$__rate_interval])) * 100)
```

⚠️ Use `=~` (regex), **not** `=`, with multi-value/All variables.

## 11.3 Repeating panels/rows

Panel options → **Repeat options** → variable `instance`: Grafana creates **one panel/row per selected host** automatically: perfect for fleet views.

## 11.4 Templated legends and titles

Title: `CPU usage: $instance`. Legend: `{{instance}}`.

---

# 1️⃣2️⃣ ANNOTATIONS: MARKING EVENTS ON GRAPHS ⭐

> **Annotations** draw vertical markers (deployments, incidents, alerts) on time-series panels so you can **correlate changes with metrics** (Part 1's "deployment markers").

| Source | How |
|---|---|
| **Built-in "Annotations & Alerts"** | Shows Grafana alert state changes |
| **Manual** | Ctrl/Cmd + click on a graph → add text/tags |
| **Query-based** | Prometheus expression, e.g., `changes(node_boot_time_seconds[1m]) > 0` → reboot markers |
| **API** | Pipelines post deployment events |

Jenkins/CI pipeline posting a deployment annotation (links to Unit III Practical 6):

```bash
# create a Service Account in Grafana (Administration → Users and access → Service accounts) with Editor role; create a token
curl -s -X POST http://<grafana>:3000/api/annotations \
  -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" \
  -d "{\"text\":\"Deploy build #${BUILD_NUMBER}\",\"tags\":[\"deploy\",\"flask-app\"]}"
```

Add an annotation query on the dashboard filtering by tag `deploy`. Now latency/error graphs show **exactly when each release happened**.

---

# 1️⃣3️⃣ BUILDING THE INFRASTRUCTURE PERFORMANCE DASHBOARD (PRACTICAL 9) ⭐⭐⭐

Goal: *design monitoring dashboards for infrastructure performance; visualize CPU, memory, storage and network utilization; analyze performance trends.*

## 13.1 Create the dashboard

```
Dashboards → New → New dashboard → (first save) name: "Infrastructure Overview (Node Exporter)" → folder "Lab" → tags: node, infra, USE
Settings: Time range "Last 3 hours" ; Auto-refresh 30s ; Timezone: Browser ; Variable: $instance (Section 11)
```

## 13.2 Panel specifications (copy each query into **Code** mode)

### Row 1: Overview (Stat / Gauge panels; query Type **Instant**)

| Panel | Type | Query | Unit | Thresholds |
|---|---|---|---|---|
| **Targets up** | Stat | `count(up{job="node"} == 1)` | none | — |
| **Targets down** | Stat | `count(up{job="node"} == 0) or vector(0)` | none | 0 green, 1 red |
| **CPU now** | Gauge | `100 - (avg(rate(node_cpu_seconds_total{mode="idle", instance=~"$instance"}[$__rate_interval])) * 100)` | Percent (0-100), min 0 max 100 | 70 yellow, 90 red |
| **Memory used** | Gauge | `(1 - sum(node_memory_MemAvailable_bytes{instance=~"$instance"}) / sum(node_memory_MemTotal_bytes{instance=~"$instance"})) * 100` | Percent (0-100) | 80 yellow, 90 red |
| **Root disk used** | Gauge | `100 - (node_filesystem_avail_bytes{instance=~"$instance", mountpoint="/", fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{instance=~"$instance", mountpoint="/", fstype!~"tmpfs|overlay"} * 100)` | Percent (0-100) | 75 yellow, 90 red |
| **Uptime** | Stat | `time() - node_boot_time_seconds{instance=~"$instance"}` | `seconds (s)` → shows d/h/m | — |
| **Load per core (5m)** | Stat | `avg(node_load5{instance=~"$instance"}) / count(node_cpu_seconds_total{mode="idle", instance=~"$instance"}) * count(count by (instance) (node_cpu_seconds_total{mode="idle", instance=~"$instance"}))` | none (2 decimals) | 0.7 yellow, 1.0 red |

(Simpler load-per-core for a single host: `node_load5 / count(node_cpu_seconds_total{mode="idle"})`.)

### Row 2: CPU (Time series)

| Panel | Query | Unit / style |
|---|---|---|
| **CPU usage %** (per host) | `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle", instance=~"$instance"}[$__rate_interval])) * 100)` | Percent (0-100), min 0, max 100, legend `{{instance}}`, threshold lines at 70/90 |
| **CPU by mode** (stacked) | `avg by (mode) (rate(node_cpu_seconds_total{mode!="idle", instance=~"$instance"}[$__rate_interval])) * 100` | Percent, **stack: normal**, legend `{{mode}}` (user, system, iowait, steal...) |
| **Load average** | A: `node_load1{instance=~"$instance"}`, B: `node_load5{...}`, C: `node_load15{...}` | short; add a threshold line at the core count |
| **I/O wait** | `avg by (instance) (rate(node_cpu_seconds_total{mode="iowait", instance=~"$instance"}[$__rate_interval])) * 100` | Percent |

### Row 3: Memory

| Panel | Query | Style |
|---|---|---|
| **Memory used %** | `(1 - node_memory_MemAvailable_bytes{instance=~"$instance"} / node_memory_MemTotal_bytes{instance=~"$instance"}) * 100` | Percent (0-100) |
| **Memory breakdown** (stacked) | A **Used**: `node_memory_MemTotal_bytes - node_memory_MemFree_bytes - node_memory_Buffers_bytes - node_memory_Cached_bytes` · B **Buffers+Cache**: `node_memory_Buffers_bytes + node_memory_Cached_bytes` · C **Free**: `node_memory_MemFree_bytes` (all with `{instance=~"$instance"}`) | `bytes (IEC)`, stacked, legend by letter names |
| **Swap used %** | `(1 - node_memory_SwapFree_bytes{instance=~"$instance"} / node_memory_SwapTotal_bytes{instance=~"$instance"}) * 100` | Percent (no data if no swap: fine) |

### Row 4: Storage

| Panel | Type / Query | Notes |
|---|---|---|
| **Filesystem used % by mount** | **Bar gauge**: `100 - (node_filesystem_avail_bytes{instance=~"$instance", fstype!~"tmpfs|overlay|squashfs|ramfs"} / node_filesystem_size_bytes{instance=~"$instance", fstype!~"tmpfs|overlay|squashfs|ramfs"} * 100)` · Instant · legend `{{instance}} {{mountpoint}}` | Percent, thresholds 75/90 |
| **Free space (GiB)** | Time series: `node_filesystem_avail_bytes{instance=~"$instance", mountpoint="/"} / 1024 / 1024 / 1024` | `gigabytes`-like unit or custom suffix " GiB" |
| **Disk throughput** | A **read**: `sum by (instance) (rate(node_disk_read_bytes_total{instance=~"$instance"}[$__rate_interval]))` · B **write**: `... node_disk_written_bytes_total ...` | `bytes/sec (IEC)` |
| **Disk IOPS** | `sum by (instance) (rate(node_disk_reads_completed_total{instance=~"$instance"}[$__rate_interval]))` and writes | `iops` |
| **Disk utilization %** | `rate(node_disk_io_time_seconds_total{instance=~"$instance"}[$__rate_interval]) * 100` | Percent; ~100 = saturated |
| **Disk fill forecast** | Stat/Table: `predict_linear(node_filesystem_avail_bytes{instance=~"$instance", mountpoint="/"}[6h], 24*3600) / 1024^3` | GiB free predicted in 24 h; red below 0 |

### Row 5: Network

| Panel | Query | Style |
|---|---|---|
| **Inbound traffic** | `sum by (instance) (rate(node_network_receive_bytes_total{instance=~"$instance", device!="lo"}[$__rate_interval])) * 8` | `bits/sec (SI)` |
| **Outbound traffic** | `sum by (instance) (rate(node_network_transmit_bytes_total{instance=~"$instance", device!="lo"}[$__rate_interval])) * 8` | `bits/sec (SI)`; mirror as negative to plot RX up/TX down: `-1 * ...` |
| **Errors and drops** | `sum by (instance) (rate(node_network_receive_errs_total{instance=~"$instance"}[$__rate_interval]) + rate(node_network_transmit_errs_total{instance=~"$instance"}[$__rate_interval]) + rate(node_network_receive_drop_total{instance=~"$instance"}[$__rate_interval]) + rate(node_network_transmit_drop_total{instance=~"$instance"}[$__rate_interval]))` | short; should stay at 0 |
| **TCP established** | `node_netstat_Tcp_CurrEstab{instance=~"$instance"}` | short |

### Row 6: Health/Status (optional)

| Panel | Query | Type |
|---|---|---|
| **Target status** | `up{job="node"}` with value mapping 1 → UP (green), 0 → DOWN (red) | **State timeline** |
| **Firing alerts** | Alert list panel, or `ALERTS{alertstate="firing"}` as Table | Shows live alerts |
| **Per-host summary** | Three instant queries (CPU %, memory %, disk %) Format Table → Merge → Organize fields; cell color by thresholds | Table |

## 13.4 Make it professional ✅

```
• Titles say WHAT + UNIT (e.g., "CPU usage (%)")      • Add a Text panel: purpose, owner, runbook link, "how to read"
• Consistent colors and thresholds across rows        • Row order follows the USE method: Utilization → Saturation → Errors
• Link panels to drill-down dashboards (Data links)  • Save with a version message; star it; set as home dashboard if useful
• Use shared tooltip and synced crosshair (dashboard settings → Graph tooltip: Shared crosshair)
```

## 13.5 Analyze trends (the "analysis" part of the practical) 🔍

```
1. Set the time range to Last 24 h / 7 d (you need data history first: leave Prometheus running a while).
2. Compare now vs yesterday: add a second query with `offset 1d`  → shows regression/seasonality
3. Generate load (Part 8: stress tool) and observe: CPU % jumps, load average rises with a lag, user-mode share grows, later returns to baseline
4. Observe memory: Used grows vs Cache; MemAvailable is the honest number
5. Observe disk: write a large file → free space drops; the forecast panel turns red; I/O throughput spikes
6. Observe network: download a file → inbound bits/s spike
7. Write 4-6 sentences: peak CPU %, time of peak, memory trend, disk growth per day, busiest network period, any anomalies and likely causes
```

---

# 1️⃣4️⃣ IMPORT A COMMUNITY DASHBOARD (FAST START) ⭐

```
Dashboards → New → Import → enter ID  1860  (Node Exporter Full, very popular) → Load
→ select data source "Prometheus" → Import
```

- **ID 1860 = "Node Exporter Full"** (grafana.com/grafana/dashboards/1860): CPU, memory, disk, network, and many system details. Always confirm the current ID/name on grafana.com, since community dashboards get updated or replaced.
- Other helpful IDs to search on grafana.com: Prometheus stats, Docker/cAdvisor, Blackbox exporter, Jenkins, Kubernetes.
- After import, the dashboard may use variables like `$job` and `$node`; ensure your Node Exporter job name matches (`node` vs `node_exporter`) or edit the variable query.
- Compare: **imported** (comprehensive, quick) vs **custom** (focused, you understand every panel). Practical 9 asks you to **design** one: build the custom dashboard (Section 13) and optionally use the imported one as a reference.

---

# 1️⃣5️⃣ A RED DASHBOARD FOR THE FLASK APP (APPLICATION VIEW) 📈

| Panel | Query | Type |
|---|---|---|
| **Request rate (req/s)** by endpoint | `sum by (endpoint) (rate(myapp_http_requests_total[$__rate_interval]))` | Time series |
| **Error ratio (%)** | `100 * sum(rate(myapp_http_requests_total{status=~"5.."}[$__rate_interval])) / sum(rate(myapp_http_requests_total[$__rate_interval]))` | Time series + Stat (thresholds 1/5) |
| **p50/p95/p99 latency** | `histogram_quantile(0.95, sum by (le) (rate(myapp_http_request_duration_seconds_bucket[$__rate_interval])))` (and 0.5, 0.99) | Time series, unit seconds |
| **Latency heatmap** | `sum by (le) (increase(myapp_http_request_duration_seconds_bucket[$__rate_interval]))` Format **Heatmap** | Heatmap |
| **Requests in flight** | `sum(myapp_http_requests_in_progress)` | Stat |
| **Orders per minute** (business) | `rate(myapp_orders_created_total[$__rate_interval]) * 60` | Time series |
| **Status code mix** | `sum by (status) (rate(myapp_http_requests_total[$__rate_interval]))` | Stacked bars / pie |
| **Deployment annotations** | Annotation query by tag `deploy` | Markers |

Link this dashboard from the infrastructure one (Data links) so you can jump **from a host problem to the app, or the reverse**.

---

# 1️⃣6️⃣ DASHBOARD DESIGN PRINCIPLES ⭐⭐⭐

## 16.1 Think in audiences and questions

| Audience | Dashboard | Key question |
|---|---|---|
| On-call engineer | Service health (golden signals), SLO status | "Is it broken? Where?" |
| SRE/platform | Infrastructure (USE), capacity, Kubernetes | "Are resources healthy? Will we run out?" |
| Developers | App RED + dependencies + deploy markers | "Did my release change behavior?" |
| Management/business | SLO attainment, orders/revenue, availability, cost | "Are we meeting goals?" |

## 16.2 Rules that make dashboards useful ✅

```
1. One dashboard = one purpose/question. Avoid 80-panel "kitchen sinks".
2. The 5-SECOND TEST: can someone tell if things are healthy within five seconds? Put the most important stats top-left.
3. Hierarchy: Overview (all services) → Service dashboard (RED) → Detail/Debug (USE, per-instance, logs/traces). Link them (drill-down).
4. Use the right method: RED for services, USE for resources, Golden Signals for general health.
5. Always show units and thresholds; consistent colors (green OK, yellow warn, red critical); same scales for comparison.
6. Use variables rather than copy-paste dashboards (host, job, environment, region).
7. Show context: deployment annotations, alert state, SLO lines, previous-day comparison.
8. Prefer percentiles/heatmaps over averages for latency; avoid pie charts for time data; limit series per graph (≤ ~10).
9. Keep queries efficient: recording rules for heavy queries, sensible time ranges and refresh rates, avoid high-cardinality panels.
10. Document: Text panel with purpose, owner, definitions, runbook links; tag and folder consistently; version in Git (dashboards as code).
```

## 16.3 Anti-patterns 🚨

| Anti-pattern | Problem | Better |
|---|---|---|
| "Wall of graphs" | Overload, nothing stands out | Prioritize; collapse detail rows |
| Averages only | Hides tail latency | p95/p99 + heatmap |
| Missing units / wrong unit | Misleading ("0.45" of what?) | Set units; percent vs ratio |
| Different time ranges/scales per panel | Hard to correlate | Use dashboard-wide time range, shared crosshair |
| Dashboards nobody owns | Rot, stale queries | Owners, review cycles, delete unused |
| Vanity metrics | No decisions | Tie panels to SLOs/questions |
| Alerts only in dashboards (eyeballing) | Missed incidents | Alert rules (Part 4), dashboards for diagnosis |
| Manual click-built prod dashboards | No history, no review | Provisioning/Git |

---

# 1️⃣7️⃣ PROVISIONING AS CODE ⭐⭐ (LINKS TO UNIT II)

Grafana can load **data sources, dashboards and alerting** from files at startup: reproducible, reviewable, GitOps-friendly.

## 17.1 Data source provisioning

`/etc/grafana/provisioning/datasources/prometheus.yml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus-main              # stable ID so dashboards can reference it
    access: proxy
    url: http://localhost:9090
    isDefault: true
    editable: false
    jsonData:
      timeInterval: 15s               # = Prometheus scrape_interval
      httpMethod: POST
```

## 17.2 Dashboard provisioning

`/etc/grafana/provisioning/dashboards/default.yml`:

```yaml
apiVersion: 1
providers:
  - name: "lab-dashboards"
    orgId: 1
    folder: "Lab"
    type: file
    disableDeletion: false
    allowUiUpdates: true
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards       # put exported dashboard JSON files here
```

Export your dashboard: **Dashboard → Share/Export → Export as JSON** (tick "Export for sharing externally" to use a data source variable), save to `/var/lib/grafana/dashboards/infra.json`, restart Grafana (`sudo systemctl restart grafana-server`). Commit JSON files to **Git**.

## 17.3 Other "dashboards as code" options

| Tool | Idea |
|---|---|
| **Terraform Grafana provider** | `grafana_folder`, `grafana_dashboard`, `grafana_data_source`, alert resources: manage Grafana like infrastructure (Unit II) |
| **Grafana Foundation SDK / Grafonnet (Jsonnet) / Grizzly** | Generate dashboard JSON from code with loops/functions (many similar services) |
| **Grafana Operator (Kubernetes)** | Dashboards/data sources as CRDs |
| **Git Sync / CI pipelines** | PR review → deploy dashboards automatically |

```hcl
# Terraform example
resource "grafana_data_source" "prometheus" {
  type = "prometheus"
  name = "Prometheus"
  url  = "http://prometheus:9090"
  is_default = true
}
resource "grafana_dashboard" "infra" {
  config_json = file("${path.module}/dashboards/infra.json")
}
```

---

# 1️⃣8️⃣ GRAFANA ALERTING ⭐

Grafana has its own **unified alerting** (rules evaluated by Grafana against **any** data source).

| Concept | Meaning |
|---|---|
| **Alert rule** | Query + condition (reduce/threshold) + evaluation interval + pending period (`for`) + labels/annotations |
| **Contact point** | Where to send: email, Slack, PagerDuty, webhook, Teams, Telegram... |
| **Notification policy** | Routing tree by labels (like Alertmanager routes): grouping, timing, mute timings |
| **Mute timings / Silences** | Scheduled or manual muting |
| **Alert states** | Normal, Pending, Alerting, NoData, Error |

Create from a panel: panel menu → **More → New alert rule** (it copies the query) → set condition (e.g., `WHEN last() OF A IS ABOVE 80`) → folder, group, evaluate every `1m` for `5m` → labels (`severity=warning`) → contact point → Save.

## 18.1 Prometheus/Alertmanager vs Grafana Alerting

| | Prometheus + Alertmanager | Grafana Alerting |
|---|---|---|
| Rules live in | YAML in Git next to Prometheus | Grafana (UI/provisioning/Terraform) |
| Data sources | Prometheus only | Many (Prometheus, Loki, SQL, CloudWatch...) |
| Reliability | Runs even if Grafana is down; battle-tested HA | Depends on Grafana availability |
| Authoring | Code | UI-friendly |
| Best for | Infrastructure/SLO alerts as code | Multi-source alerts, teams who live in Grafana; can also **manage** Prometheus/Mimir rules |

Practical 8 uses **Prometheus rules** (Part 4). In Practical 9 you can optionally add one Grafana alert to compare.

---

# 1️⃣9️⃣ USERS, PERMISSIONS AND SECURITY 🔐

| Concept | Notes |
|---|---|
| **Org roles** | **Viewer** (view), **Editor** (edit dashboards), **Admin** (manage org/data sources/users) |
| **Server Admin** | Instance-wide admin (the `admin` user) |
| **Teams & folder permissions** | Grant teams access to folders/dashboards |
| **Auth options** | Local users, LDAP, OAuth (Google, GitHub, Azure AD), SAML (Enterprise), anonymous (avoid) |
| **Service accounts and tokens** | Replace the old "API keys" for automation (annotations, provisioning, CI) |
| **Hardening checklist** | |

```
☐ Change the default admin password immediately (or set GF_SECURITY_ADMIN_PASSWORD at deploy)
☐ Disable anonymous access and public sign-up (allow_sign_up = false)
☐ Serve over HTTPS (Nginx/ALB/Caddy reverse proxy or built-in cert); restrict port 3000 in the Security Group
☐ Use SSO/OAuth for teams; least-privilege roles; folder-level permissions
☐ Store secrets (data source passwords, SMTP) via env vars or secret managers; protect grafana.ini
☐ Keep Grafana and plugins updated; install only trusted plugins (signed)
☐ Back up the Grafana DB (/var/lib/grafana/grafana.db) or use PostgreSQL; keep dashboards in Git
```

---

# 2️⃣0️⃣ A DOCKER COMPOSE LAB STACK (OPTIONAL ALTERNATIVE TO PART 8 INSTALLS) 🐳

`docker-compose.yml`:

```yaml
services:
  prometheus:
    image: prom/prometheus            # pin a specific version in real use
    ports: ["9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./rules:/etc/prometheus/rules:ro
      - prom-data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=15d
      - --web.enable-lifecycle
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter
    pid: host                         # see host processes
    network_mode: host                # report host network interfaces (listens on :9100 of the host)
    volumes:
      - /:/host:ro,rslave
    command:
      - --path.rootfs=/host
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager
    ports: ["9093:9093"]
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    restart: unless-stopped

  grafana:
    image: grafana/grafana
    ports: ["3000:3000"]
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=ChangeMe-Strong-1!      # use a secret in real life
    volumes:
      - grafana-data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning:ro
    depends_on: [prometheus]
    restart: unless-stopped

volumes:
  prom-data:
  grafana-data:
```

Notes: with `network_mode: host` on node-exporter, Prometheus (in the bridge network) scrapes it via the **host's IP** (e.g., `host.docker.internal` on Docker Desktop, or the EC2 private IP), not `localhost`. In `prometheus.yml` use `targets: ["<host-private-ip>:9100"]`. In Grafana's data source use `http://prometheus:9090` (service name). Bring up: `docker compose up -d`.

---

# 2️⃣1️⃣ TROUBLESHOOTING TABLE 🛠️

| Symptom | Cause | Fix |
|---|---|---|
| Can't open `:3000` | Service down / Security Group / firewall | `systemctl status grafana-server`; open 3000 to your IP |
| `grafana-server` fails to start | Port in use, config syntax error | `journalctl -u grafana-server`; check `grafana.ini` |
| Data source "Save & test" fails: *connection refused* | Wrong URL/port; Prometheus not running; `localhost` inside a container | Use correct host (private IP, Docker service name); `curl` from the Grafana host |
| *Bad Gateway* / *context deadline exceeded* | Firewall/SG between hosts; slow Prometheus | Open 9090 from Grafana's IP/SG; check Prometheus health |
| Panel **"No data"** | Wrong metric/label/job; empty time range; target down; variable returns nothing | Run the query in Prometheus/Explore; check `up`; verify `job`/`instance` labels; widen time range |
| Values 100× too big/small | Percent unit mismatch (0-1 vs 0-100) | Choose Percent (0-1.0) or multiply by 100 consistently |
| Graph flat/jumpy | `rate` window too small/large | Use `$__rate_interval`; set data source Scrape interval to 15s |
| Variable dropdown empty | Wrong `label_values` query/data source | Test query in Explore; use `label_values(up{job="node"}, instance)` |
| `All` shows nothing / only one instance | Used `=` instead of `=~` | `instance=~"$instance"` |
| Imported dashboard shows N/A | Job/label names differ (`job="node_exporter"`) or datasource not mapped | Edit variables/queries; select correct data source on import |
| Dashboard JSON import error | Data source UID mismatch | Re-map data source; export "for sharing externally" |
| Annotations don't appear | Wrong tag filter/token/permissions | Check API response, token role (Editor), annotation query tags |
| Provisioned dashboard not updating | `allowUiUpdates`/`updateIntervalSeconds`, file ownership | Fix permissions (`grafana` user), restart service |
| Login loop/forgot admin password | Credentials changed | `sudo grafana cli admin reset-admin-password '<new>'` |
| Alert doesn't fire/notify | Contact point/policy not set; `NoData` handling | Check Alerting → state history, test contact point |

---

# 2️⃣2️⃣ VIVA QUESTIONS (PART 5) 🎯

**Q1. What is Grafana and how is it different from Prometheus?** Grafana is a visualization and alerting platform that queries data sources; Prometheus collects and stores metrics. Grafana doesn't store the metrics.

**Q2. What is a data source?** A configured connection (URL, auth) that tells Grafana where and how to query data, e.g., Prometheus.

**Q3. Default Grafana port and credentials?** Port 3000; `admin/admin` (must be changed at first login).

**Q4. What is a panel? A dashboard? A row?** Panel = one visualization; dashboard = a collection of panels with settings; row = collapsible group of panels.

**Q5. When would you use Stat, Gauge, Time series and Table?** Stat for a single key number; Gauge for value vs thresholds; Time series for trends; Table for exact multi-series details.

**Q6. What is `$__rate_interval`?** A Grafana variable that gives a `rate()` window adapted to scrape interval and zoom, avoiding flat or empty graphs.

**Q7. What are dashboard variables and why use them?** Dropdowns (e.g., `$instance` from `label_values(...)`) that parametrize queries so one dashboard serves many hosts/jobs; use `=~` for multi-value.

**Q8. What are transformations?** Client-side operations to reshape, merge, filter and calculate on query results.

**Q9. What are annotations?** Time markers (deployments, incidents, alerts) shown on graphs to correlate events with metrics.

**Q10. How do you visualize CPU, memory, storage and network for a host?** Using Node Exporter metrics via PromQL: CPU `100 - avg(rate(idle))*100`, memory `(1 - MemAvailable/MemTotal)*100`, disk used % from filesystem metrics, network `rate(bytes_total)*8`.

**Q11. How do you provision Grafana as code?** YAML files for data sources and dashboard providers plus dashboard JSON in provisioning directories, or Terraform/Grafonnet; stored in Git.

**Q12. Grafana Alerting vs Prometheus/Alertmanager?** Grafana evaluates rules against many data sources from its UI; Prometheus rules are code, run independently of Grafana and route via Alertmanager.

**Q13. What is the "5-second test" for dashboards?** Whether a viewer can judge system health within five seconds; put key stats first.

**Q14. How do you secure Grafana?** Change default password, disable anonymous/sign-up, HTTPS, SSO, least-privilege roles, service accounts for automation, restrict port, keep updated.

**Q15. How would you import a ready-made Node Exporter dashboard?** Dashboards → Import → ID 1860 (Node Exporter Full) → select the Prometheus data source.

---

# 🧠 PART 5 MASTER SHEET

```
GRAFANA: open-source visualization/observability platform ; queries data sources (does not store metrics) ; port 3000 ; admin/admin → change ; AGPLv3 ; LGTM stack
PATHS: /etc/grafana/grafana.ini · provisioning/{datasources,dashboards,alerting} · /var/lib/grafana · /var/log/grafana ; env GF_<SECTION>_<KEY>
INSTALL: apt repo (gpg-full.key) → apt install grafana → systemctl enable --now grafana-server ; or Docker grafana/grafana
DATA SOURCE: Connections → Data sources → Prometheus → URL http://host:9090 → Scrape interval 15s → Save & test
PANELS: Time series (trends) | Stat/Gauge (now) | Bar gauge | Table | Heatmap | State timeline | Alert list | Text
QUERY: Code mode PromQL ; Range vs Instant ; Format ; legend {{instance}} ; $__rate_interval · $__interval · $__range
FIELD CONFIG: unit (percent 0-100 vs 0-1, bytes IEC, bits/sec, seconds) · thresholds green/yellow/red · min/max · mappings · overrides · stacking
VARIABLES: label_values(metric, label) · multi-value + All · use =~ · chained variables · repeat panels/rows
TRANSFORMATIONS: merge, organize fields, reduce, filter, join ; ANNOTATIONS: deployments/alerts, API with service-account token
DASHBOARD (P9): Overview stats · CPU (usage, by mode, load) · Memory (used %, breakdown, swap) · Storage (used %, throughput, IOPS, util, forecast) · Network (in/out bits/s, errors, TCP)
IMPORT: Node Exporter Full ID 1860 ; DESIGN: one purpose, 5-second test, USE/RED, units, drill-down, variables, docs, Git
PROVISIONING: datasource YAML + dashboard provider + JSON ; Terraform grafana provider ; Grafonnet/Foundation SDK
GRAFANA ALERTING: rules + contact points + notification policies + mute timings (many sources) vs Prometheus rules + Alertmanager (code, resilient)
SECURITY: change password, no anonymous, HTTPS, SSO, RBAC, service accounts, restrict 3000, update, back up DB
```

---

## 🫡 PART 5 STATUS

You can install Grafana, connect Prometheus, build and parametrize dashboards (CPU, memory, storage, network and RED), apply design principles, provision dashboards as code, and compare Grafana and Prometheus alerting.

**STOP here.** Next up:

> **PART 6 — Scaling Monitoring Solutions (HA, federation, sharding, remote write, Thanos/Mimir/VictoriaMetrics, cardinality control, Kubernetes, managed/cloud monitoring) 📈**
