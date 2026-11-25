# 🏆 UNIT IV — GRAND CONSOLIDATION

## Monitoring and Observability — From Scratch → End-to-End

> **Covered in this document:** the whole unit in one sentence → the complete story → part-by-part revision (Parts 1-8) → master comparison tables (monitoring vs observability, pillars, golden signals/RED/USE, push vs pull, metric types, Prometheus vs Alertmanager vs Grafana, alerting concepts, scaling options, Loki vs ELK, Thanos vs Mimir vs VictoriaMetrics, self-managed vs managed) → cheat sheets (config skeletons, PromQL, alert rule, Alertmanager, Grafana, CLI/API commands) → decision tables → common errors table → monitoring security checklist → lab ideas → 3 real-world scenarios → 30 best viva questions with answers → common traps → what to write (and not write) in an exam → master sheet → memory maps → how Unit IV connects to Units I, II, III, V and VI.

---

# 🚀 THE ENTIRE UNIT IV IN ONE SENTENCE

> **Monitoring tells you *that* something is wrong and observability helps you find out *why*: Prometheus scrapes metrics (from exporters like Node Exporter) into a time-series database queried with PromQL, rules and Alertmanager turn symptoms into routed, grouped, actionable alerts, Grafana turns the data into performance dashboards, and at scale you add HA, sharding, remote storage (Thanos/Mimir), logs (Loki), traces (OpenTelemetry/Tempo), SLO-based alerting and strict cost and cardinality control.**

---

# 1️⃣ THE COMPLETE UNIT IV STORY

```
WHY             Fast delivery (Unit III) without visibility = crashes ; monitoring = known questions, observability = unknown questions     [Part 1]
WHAT TO MEASURE Three pillars (metrics, logs, traces) ; Golden Signals / RED (services) / USE (resources) ; SLI → SLO → SLA ; error budget      [Part 1]
COLLECT         Prometheus: pull model, TSDB, labels, 4 metric types, exporters (Node Exporter :9100), service discovery, instrumentation      [Part 2]
QUERY           PromQL: selectors, rate(), aggregation by/without, histogram_quantile, predict_linear, recording rules                          [Part 3]
ALERT           Alert rules (expr, for, labels, annotations) → Alertmanager (group, route, inhibit, silence, notify) ; SLO burn-rate alerts     [Part 4]
VISUALIZE       Grafana: data sources, panels, variables, thresholds, annotations, provisioning, dashboards (USE/RED), Grafana alerting        [Part 5]
SCALE           Optimize cardinality → HA pairs → sharding → federation → remote write → Thanos/Mimir/VM → Kubernetes Operator → managed      [Part 6]
GO DEEPER       Structured logs + Loki/LogQL, tracing + OpenTelemetry, correlation, SLO alerting, synthetic/RUM, eBPF, AIOps, chaos           [Part 7]
PROVE IT        P8: Prometheus + Node Exporter + alert rules + validated alerts ; P9: Grafana + dashboards + trend analysis                     [Part 8]
```

---

# 2️⃣ PART-BY-PART REVISION

## Part 1: Monitoring and observability from scratch
```
Monitoring = predefined metrics/checks/alerts ("Is it broken?") ; Observability = infer internal state from telemetry ("Why?") ; monitoring ⊂ observability
Pillars: Metrics (numbers over time) | Logs (events) | Traces (request path across services) + events, profiles ; correlation is the superpower
Pipeline: instrument → collect → process → store → query → visualize → alert → act
Methods: Four Golden Signals (Latency, Traffic, Errors, Saturation) | RED (Rate, Errors, Duration) for services | USE (Utilization, Saturation, Errors) for resources
SLI (measure) → SLO (target) → SLA (contract) ; error budget = 100% − SLO ; 99.9% = 43.8 min/month ; 99.99% = 4.38 min/month
Black-box (outside, symptoms) vs White-box (inside) ; Pull (Prometheus) vs Push (StatsD, Pushgateway, OTel)
Alerts: actionable, urgent, symptom-based ; alert fatigue ; MTTD, MTTA, MTTR, MTBF ; percentiles not averages
```

## Part 2: Prometheus fundamentals
```
Prometheus: open source (SoundCloud 2012), CNCF graduated, Go, pull-based, TSDB + PromQL + Alertmanager ; 3.x line with LTS releases
Architecture: scrape manager + service discovery → TSDB (head + WAL + 2h blocks) → PromQL/HTTP API ← Grafana ; rules → Alertmanager ; Pushgateway ; remote write
Data model: metric_name{labels} value @ms ; series = name + ALL labels ; auto labels job, instance ; naming snake_case, base units, counters _total
Types: Counter (rate) | Gauge | Histogram (_bucket{le}, _sum, _count; aggregatable) | Summary (client quantiles; not aggregatable)
Config: global, rule_files, alerting, scrape_configs ; relabel_configs (targets) vs metric_relabel_configs (samples) ; promtool check ; reload (SIGHUP, /-/reload)
up (1/0) ; retention 15d ; disk = retention × samples/s × ~1-2 B ; cardinality = danger (no user_id labels)
Exporters: node_exporter (CPU/mem/disk/net), blackbox (probes), cAdvisor, DB exporters, textfile collector ; Pushgateway only for batch jobs
Instrument: prometheus_client Counter/Gauge/Histogram + /metrics ; route templates ; define metrics once
```

## Part 3: PromQL
```
Types: instant vector | range vector | scalar | string ; instant vs range query ; staleness 5m
Selectors: = != =~ !~ ; [5m] ; offset ; @
Operators: arithmetic ; comparison filters (bool) ; and/or/unless ; on/ignoring/group_left (label mismatch = EMPTY result!)
Aggregation: sum avg min max count topk quantile ; by (keep) / without (drop) ; rate FIRST, then sum
Functions: rate (default) | irate (spiky) | increase | delta/deriv | predict_linear | *_over_time | absent | changes/resets | clamp | label_replace
Histograms: histogram_quantile(0.95, sum by (le, ...) (rate(x_bucket[5m]))) ; avg = rate(_sum)/rate(_count) ; never average percentiles
Window ≥ 4 × scrape_interval ; Grafana $__rate_interval ; recording rules: level:metric:operations
Node cookbook: CPU% = 100 - avg(rate(idle))*100 | MEM% = (1 - MemAvailable/MemTotal)*100 | DISK% = 100 - avail/size*100 | NET = rate(bytes_total)*8
               LOAD/core = node_load1 / on(instance) count by (instance)(cpu idle) | UPTIME = time() - node_boot_time_seconds
```

## Part 4: Alerting
```
Prometheus = WHEN (rules) ; Alertmanager = WHO/HOW (:9093, cluster :9094)
Rule: alert, expr, for, keep_firing_for, labels (route), annotations (summary, description, runbook_url) ; templates $labels, $value
States: inactive → pending (for) → firing → resolved ; ALERTS metric ; Watchdog heartbeat
Alertmanager: dedupe, group (group_by), route (matchers tree, continue), inhibit (source/target/equal), silence, notify (send_resolved), HA gossip
Timings: group_wait 30s | group_interval 5m | repeat_interval 4h ; receivers: email, Slack, PagerDuty, webhook, SNS ; secrets in *_file
Tools: promtool check/test rules ; amtool alert/silence/check-config/config routes test
SLO alerts: burn rate = error ratio ÷ (1−SLO) ; multi-window 14.4× (1h&5m), 6× (6h&30m), 1× (3d&6h)
Best practice: symptoms not causes, actionable, runbooks, `for`, group/inhibit/route, review noisy alerts, rules as code
```

## Part 5: Grafana
```
Grafana: visualization platform (queries data sources; stores no metrics) ; port 3000 ; admin/admin → change ; LGTM stack
Install: apt repo → grafana-server ; data source: Prometheus http://host:9090 + scrape interval 15s
Panels: Time series | Stat | Gauge | Bar gauge | Table | Heatmap | State timeline | Alert list ; Code-mode PromQL ; legend {{instance}} ; Range vs Instant
Units (percent 0-100 vs 0-1, bytes IEC, bits/sec, seconds) ; thresholds green/yellow/red ; mappings ; overrides ; stacking
Variables: label_values(...) ; multi + All ; use =~ ; repeat panels ; annotations (deployments, alerts) ; transformations
Dashboard (P9): Overview + CPU + Memory + Storage + Network ; import Node Exporter Full (ID 1860) ; design: 5-second test, USE/RED, drill-down
Provisioning: datasource YAML + dashboard provider + JSON ; Terraform grafana provider ; Grafana Alerting (many sources) vs Prometheus rules (code, resilient)
```

## Part 6: Scaling monitoring
```
Dimensions: series (RAM) | samples/s | queries | retention (disk) | availability | geography/teams
Capacity: samples/s = series/interval ; disk = retention × samples/s × ~1.5 B ; RAM ≈ series × ~3-6 KB (example: 3M series @30s, 15d → 100k/s, ~194 GB, ~12 GB)
Ladder: optimize (drop labels/metrics, sample_limit, recording rules, query limits) → HA pair (external label replica, Alertmanager cluster, list ALL AMs)
        → functional sharding → hashmod sharding → federation (aggregates only) → remote_write + Agent mode → Thanos / Mimir / VictoriaMetrics / managed
Thanos: sidecar + store gateway + querier (dedup) + compactor (downsample) + ruler/receive ; object storage ; Mimir: multi-tenant, remote_write, billions of series
Kubernetes: kube-prometheus-stack (Operator) ; node-exporter + cAdvisor + kube-state-metrics ; CRDs ServiceMonitor/PodMonitor/PrometheusRule
Cloud: CloudWatch (+agent), AMP/AMG, Azure Monitor/Managed Prometheus/Grafana, Google Managed Prometheus
Cost: cardinality, filtering, downsampling, trace sampling, log hygiene ; Meta-monitoring: Watchdog + external dead-man switch + independent monitor
```

## Part 7: Advanced observability
```
Strategy: SLO-first · instrument once (OpenTelemetry) · route anywhere (Collector) · correlate · alert on burn · govern cost/privacy
Logs: structured JSON, trace_id, no PII ; Loki (label-only index, LogQL) vs Elasticsearch (full-text)
Tracing: trace/spans, W3C traceparent, sampling (head/tail), Jaeger (Uber) / Tempo ; OpenTelemetry: API + SDK + Collector (receivers → processors → exporters), OTLP 4317/4318
Correlation: trace_id in logs, exemplars, Grafana derived fields, shared labels, deploy annotations
SLO observability: SLIs/SLOs, burn-rate alerts, Sloth/Pyrra ; synthetic + RUM (Core Web Vitals LCP/INP/CLS) ; profiling (Pyroscope) ; eBPF ; service mesh
AIOps: z-score PromQL, seasonality (offset 1w), ML, alert correlation, LLM helpers (+ risks) ; canary analysis (Argo Rollouts/Flagger) ; chaos engineering, game days
Incidents + ODD: detect → triage → diagnose → mitigate → communicate → postmortem ; observability in Definition of Done ; governance: PII, sampling, retention, RBAC, cost
```

## Part 8: Practicals 8 and 9
```
P8: EC2 → users → Node Exporter → Prometheus (systemd) → prometheus.yml + promtool check → Targets UP → PromQL (CPU/mem/disk/net) → Alertmanager + webhook receiver
    → alerts.yml + promtool test → stress-ng/fallocate/stop exporter → pending → firing → webhook → resolved → silence demo
P9: Install Grafana → Prometheus data source → import Node Exporter Full + lab JSON → build dashboard (variable + CPU/memory/storage/network) → load tests → trend analysis
```

---

# 3️⃣ MASTER COMPARISON TABLES

## 3.1 Monitoring vs Observability

| | Monitoring | Observability |
|---|---|---|
| Question | Is it working? | Why isn't it working? |
| Failure type | Known | Unknown/novel |
| Data | Predefined metrics/checks | Correlated metrics + logs + traces (+ events/profiles) |
| Output | Alerts, dashboards | Understanding, root cause |
| Relationship | Part of observability | Superset |

## 3.2 Three pillars

| | Metrics | Logs | Traces |
|---|---|---|---|
| Data | Numbers over time | Timestamped events | Request journey (spans) |
| Cost | Low | High | Medium (sampled) |
| Best for | Alerting, trends, capacity | Debug detail, audit | Latency, dependencies |
| Query | PromQL | LogQL / Lucene | TraceQL / trace search |
| Tools | Prometheus | Loki, ELK | Jaeger, Tempo, OpenTelemetry |

## 3.3 Golden Signals vs RED vs USE

| | Four Golden Signals | RED | USE |
|---|---|---|---|
| Letters | Latency, Traffic, Errors, Saturation | Rate, Errors, Duration | Utilization, Saturation, Errors |
| For | Any user-facing system | Request-driven services | Resources (CPU, memory, disk, network) |
| Example | p95 latency, req/s, 5xx %, queue depth | per-endpoint rate/errors/p95 | CPU %, load, memory, disk %, I/O wait |

## 3.4 Pull vs Push

| | Pull (Prometheus) | Push |
|---|---|---|
| Initiator | Server scrapes targets | Targets send data |
| Health signal | `up` / failed scrape | Silence is ambiguous |
| Best for | Long-running services | Short-lived jobs, edge, serverless |
| Tools | Prometheus | StatsD, OTel exporters, Pushgateway |

## 3.5 Metric types

| | Counter | Gauge | Histogram | Summary |
|---|---|---|---|---|
| Moves | Up only | Up/down | Cumulative buckets | Client quantiles |
| Query | `rate()`/`increase()` | Direct | `histogram_quantile(rate(_bucket))` | Read quantile |
| Aggregatable | Yes | Yes | **Yes** | **No** |

## 3.6 Who does what (the stack)

| Component | Role | Port |
|---|---|---|
| Node Exporter | Host metrics | 9100 |
| Prometheus | Scrape, store, query, evaluate rules | 9090 |
| Alertmanager | Group, route, silence, notify | 9093 (cluster 9094) |
| Grafana | Dashboards, exploration, alerting UI | 3000 |
| Pushgateway | Batch job metrics | 9091 |
| Blackbox exporter | External probes | 9115 |
| OTel Collector (OTLP) | Telemetry routing | 4317 gRPC / 4318 HTTP |
| Loki / Tempo | Logs / traces | 3100 / 3200 |

## 3.7 Alerting concepts

| | Inhibition | Silence |
|---|---|---|
| Trigger | Automatic (source alert firing) | Manual |
| Defined in | `alertmanager.yml` | UI/API/`amtool` |
| Use | Suppress redundant alerts | Maintenance windows |

| Alert state | Meaning |
|---|---|
| Inactive | Condition false |
| Pending | True, waiting for `for:` |
| Firing | Sent to Alertmanager |
| Resolved | Cleared; "resolved" notification |

## 3.8 PromQL essentials

| Need | Use |
|---|---|
| Per-second rate of a counter | `rate(x[5m])` |
| Total increase in a window | `increase(x[1h])` |
| Spiky instant rate (graphs only) | `irate(x[2m])` |
| Percentile | `histogram_quantile(0.95, sum by (le) (rate(x_bucket[5m])))` |
| Forecast a gauge | `predict_linear(x[6h], 4*3600)` |
| Existence | `absent(x)` |
| Keep labels / drop labels | `by (...)` / `without (...)` |

## 3.9 Scaling options

| | Plain Prometheus | Federation | Thanos | Mimir/Cortex | VictoriaMetrics | Managed |
|---|---|---|---|---|---|---|
| Long-term storage | Local disk | No | Object storage | Object storage | Efficient disks | Included |
| Global query | No | Aggregates only | Yes | Yes | Yes (cluster) | Yes |
| HA dedup | Manual pair | — | Yes | Yes | Yes | Yes |
| Multi-tenancy | No | No | Limited | Strong | Cluster | Yes |
| Ops effort | Low | Low | Medium | Medium-High | Low-Medium | Lowest |

## 3.10 Loki vs Elasticsearch

| | Loki | Elasticsearch |
|---|---|---|
| Index | Labels only | Full-text |
| Cost | Low | Higher |
| Search power | Good for known streams | Excellent |
| Best for | Ops logs with Grafana | Search/analytics/SIEM |

## 3.11 Self-managed vs managed vs cloud-native

| | Self-managed Prometheus stack | Managed Prometheus/Grafana | CloudWatch/Azure Monitor |
|---|---|---|---|
| Control | Max | Medium | Limited |
| Ops burden | High | Low | Lowest |
| Portability | High | High | Low |
| Best for | Platform teams, multi-cloud | No-ops Prometheus | Cloud resource coverage |

## 3.12 Grafana vs Prometheus alerting

| | Prometheus + Alertmanager | Grafana Alerting |
|---|---|---|
| Rules as | YAML in Git | UI/provisioned |
| Data sources | Prometheus | Many |
| Resilience | Independent of Grafana | Depends on Grafana |

---

# 4️⃣ CHEAT SHEETS ⌨️

## 4.1 Prometheus config skeleton

```yaml
global: { scrape_interval: 15s, evaluation_interval: 15s, external_labels: { env: dev } }
alerting: { alertmanagers: [ { static_configs: [ { targets: ["localhost:9093"] } ] } ] }
rule_files: ["/etc/prometheus/rules/*.yml"]
scrape_configs:
  - job_name: node
    static_configs: [ { targets: ["localhost:9100"] } ]
```

## 4.2 Alert rule template

```yaml
groups:
  - name: example
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels: { severity: warning }
        annotations: { summary: "High CPU on {{ $labels.instance }}", runbook_url: "https://wiki/runbook" }
```

## 4.3 Alertmanager skeleton

```yaml
route: { receiver: default, group_by: [alertname, instance], group_wait: 30s, group_interval: 5m, repeat_interval: 4h,
         routes: [ { matchers: ['severity="critical"'], receiver: oncall } ] }
inhibit_rules: [ { source_matchers: ['alertname="InstanceDown"'], target_matchers: ['severity="warning"'], equal: [instance] } ]
receivers: [ { name: default }, { name: oncall, webhook_configs: [ { url: "http://host/alerts", send_resolved: true } ] } ]
```

## 4.4 Must-know PromQL (copy-ready)

```promql
up == 0
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
100 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"} * 100)
rate(node_network_receive_bytes_total{device!="lo"}[5m]) * 8
node_load1 / on(instance) count by (instance) (node_cpu_seconds_total{mode="idle"})
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 4*3600) < 0
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

## 4.5 Commands

```bash
# services
sudo systemctl status|restart|reload prometheus ; sudo journalctl -u prometheus -f
promtool check config prometheus.yml ; promtool check rules rules/*.yml ; promtool test rules tests/alerts_test.yml
curl -X POST localhost:9090/-/reload                       # needs --web.enable-lifecycle
amtool check-config alertmanager.yml ; amtool config routes test --config.file=alertmanager.yml severity=critical
amtool silence add alertname=X --duration=2h --comment="why" ; amtool alert ; amtool silence query
# APIs
curl -s localhost:9090/api/v1/query --data-urlencode 'query=up' | jq
curl -s localhost:9090/api/v1/alerts | jq ; curl -s localhost:9093/api/v2/alerts | jq ; curl -s localhost:9090/api/v1/targets | jq
# load tests (lab)
stress-ng --cpu 0 --timeout 300s ; stress-ng --vm 1 --vm-bytes 1G --vm-keep --timeout 300s ; sudo fallocate -l 7G /var/tmp/fill.img
```

## 4.6 Grafana quick settings

```
Data source: http://localhost:9090, Scrape interval 15s → Save & test
Variable:    label_values(up{job="node"}, instance)  (multi + All) → use instance=~"$instance"
Range window: $__rate_interval ; Units: percent (0-100) vs percentunit (0-1) ; thresholds green/yellow/red ; import ID 1860
```

---

# 5️⃣ DECISION TABLES 🎯

## 5.1 Which metric type?

| Need | Type |
|---|---|
| Counts that only increase (requests, errors, bytes) | Counter (name `_total`, query with `rate`) |
| Current value (memory, queue, temperature) | Gauge |
| Latency/size distribution across instances | Histogram |
| Exact single-process quantiles | Summary |

## 5.2 Which monitoring method?

| Question | Method |
|---|---|
| Are my services healthy for users? | RED / Golden Signals |
| Are my machines healthy? | USE |
| Is the site reachable from outside? | Black-box/synthetic probes |
| How do real users experience it? | RUM (Core Web Vitals) |
| Why is it slow? | Traces + logs + profiles |

## 5.3 Which signal for which problem?

| Problem | Start with |
|---|---|
| Something is wrong / alert | Metrics (dashboards, SLO burn) |
| Which service/dependency is slow | Traces |
| Why a specific request failed | Logs (filtered by trace_id) |
| Which function burns CPU | Profiles |

## 5.4 What to do when Prometheus struggles?

| Symptom | First action | Then |
|---|---|---|
| High RAM/OOM | Find/drop high-cardinality labels, set `sample_limit` | Shard / Agent + central store |
| Slow dashboards | Recording rules, shorter ranges | Query frontend/caching |
| Need long retention | Bigger disk | Thanos/Mimir + object storage |
| Need global view | Federate aggregates | Thanos Querier / Mimir |
| SPOF | 2 replicas + AM cluster | Global HA layer |

## 5.5 Prometheus alerts vs Grafana alerts

```
Infrastructure/SLO alerts as code, must work even if Grafana is down → Prometheus rules + Alertmanager
Alerts across several data sources or managed in the UI by app teams  → Grafana Alerting
```

---

# 6️⃣ COMMON ERRORS TABLE 🛠️

| Symptom | Cause | Fix |
|---|---|---|
| Target DOWN: connection refused | Exporter not running/wrong port | `systemctl status`, `curl :9100/metrics` |
| Target DOWN: context deadline exceeded | SG/firewall/slow target | Open port from Prometheus SG; raise timeout |
| Config not applied | No reload | `/-/reload` or restart; `promtool check config` |
| Alertmanager won't start: no private IP | Gossip needs address | `--cluster.listen-address=` (empty) for single node |
| Alert stuck pending | `for:` not satisfied | Keep condition true; check expr |
| Alert never appears | Query returns nothing (label mismatch, wrong metric) | Test expr in Query tab |
| **Binary operation returns empty** | Label sets differ | Use `on(instance)` / `ignoring(...)` |
| Firing but no notification | Prometheus↔Alertmanager link, route/silence/inhibit, receiver down | `/api/v1/alertmanagers`, `amtool config routes test`, receiver logs |
| `rate()` empty/jumpy | Window too small | ≥ 4× scrape interval; `$__rate_interval` |
| Raw counter graphed | Missing `rate()` | `rate()`/`increase()` |
| Percentile NaN | `le` dropped in `sum by` | `sum by (le, ...)` |
| Grafana "No data" | Wrong job/label, target down, time range | Check `up`, labels, range |
| Grafana datasource connection refused | Wrong URL/`localhost` in container | Use correct host/service name |
| Values 100× off | Percent vs ratio unit | Match unit to the query |
| Variable "All" shows nothing | `=` instead of `=~` | `instance=~"$instance"` |
| Disk full on Prometheus host | Retention vs disk | Set retention size/time, bigger disk, remote storage |
| Memory explosion | High cardinality | Drop labels/metrics, `sample_limit` |

---

# 7️⃣ MONITORING SECURITY CHECKLIST 🔐 (BRIDGE TO UNIT V)

```
☐ Prometheus/Alertmanager/Node Exporter have no auth by default → restrict Security Groups to trusted IPs; never open to 0.0.0.0/0
☐ Put TLS + authentication (reverse proxy/--web.config.file/OAuth) in front of UIs; separate service users (nologin); least privilege
☐ Grafana: change admin password, disable anonymous/sign-up, HTTPS, SSO, RBAC, service accounts for automation, keep updated
☐ Secrets (SMTP, Slack, PagerDuty, DB) in *_file/secret managers; never in Git; rotate
☐ No PII/secrets in metric labels or logs; redact in the Collector; retention and access control for logs/traces
☐ Alertmanager can silence everything: protect it; audit silences/config changes
☐ Object storage buckets private, encrypted, IAM roles (not keys); per-tenant limits; patch components (CVEs exist)
☐ Meta-monitoring + backups (Grafana DB, dashboards/rules in Git)
```

---

# 8️⃣ LAB IDEAS BEYOND THE SYLLABUS

| Lab | Idea |
|---|---|
| A | Monitor the **Jenkins** server (Prometheus plugin) and build durations (Unit III) |
| B | Instrument the **Flask CI/CD app** and add RED + SLO burn-rate alerts; use Jenkins to post Grafana annotations |
| C | Add **Blackbox exporter** probes for your site and TLS-expiry alert |
| D | Provision Prometheus/Grafana EC2 stack with **Terraform/CloudFormation** (Unit II) and Grafana dashboards via the Terraform provider |
| E | Docker Compose stack (Prometheus, Node Exporter, Alertmanager, Grafana, cAdvisor) |
| F | Add **Loki** + a log agent and correlate logs with metrics in Grafana |
| G | Add **OpenTelemetry** tracing to the Flask app with a Collector + Tempo/Jaeger |
| H | Two Prometheus replicas + Alertmanager cluster; verify de-duplication |
| I | **Thanos sidecar** to S3 and query 2 hours of historical data |
| J | `kube-prometheus-stack` on a local Kubernetes (kind/minikube) + a `ServiceMonitor` |
| K | Chaos test: kill a container/instance and verify alert + runbook within the SLO |

---

# 9️⃣ THREE REAL-WORLD SCENARIOS

## Scenario A: "Customers report slowness before we notice"
```
Problem: only host CPU alerts; no user-facing signals
Fix: RED metrics + p95 latency alerts, SLOs with burn-rate alerts, synthetic probes from outside, Grafana service dashboard with deploy annotations,
     runbooks in alert annotations ; MTTD drops from tens of minutes to minutes
Concepts: Golden Signals/RED, SLO alerting, black-box + white-box (Parts 1, 4, 7)
```

## Scenario B: "Prometheus keeps running out of memory after we moved to Kubernetes"
```
Problem: pod churn and unbounded labels create millions of series
Fix: find offenders (TSDB status), drop pod_template_hash/uid labels and noisy metrics, sample_limit per job, recording rules for dashboards,
     per-cluster Prometheus (Operator) with remote write to Mimir/Thanos for long retention and global queries
Concepts: cardinality, relabeling, sharding, remote write (Parts 2, 6)
```

## Scenario C: "Pager fatigue: 200 alerts a week, few are real"
```
Problem: cause-based thresholds, no grouping/inhibition, no `for`, no owners
Fix: move to symptom/SLO alerts, add `for:` and keep_firing_for, group_by + inhibition (node down mutes its resource alerts), route by team/severity,
     runbooks, weekly alert review to delete or tune noisy rules, track pages per shift
Concepts: alert design, Alertmanager routing, error budgets (Parts 1, 4)
```

---

# 🔟 30 BEST UNIT IV VIVA QUESTIONS 🔥 (WITH ANSWERS)

**Foundations**

1. **Monitoring vs observability?** Monitoring checks predefined metrics for known failures ("is it working?"); observability lets you ask new questions from correlated telemetry ("why?"); monitoring is part of observability.
2. **Three pillars?** Metrics, logs, traces (plus events and profiles).
3. **Four Golden Signals; RED; USE?** Latency/traffic/errors/saturation; rate/errors/duration for services; utilization/saturation/errors for resources.
4. **SLI, SLO, SLA, error budget?** Measured indicator; internal target; customer contract; 100% − SLO as allowed unreliability.
5. **Push vs pull?** Pull: server scrapes targets (Prometheus), failed scrape = down; push: targets send data (StatsD/OTel/Pushgateway), suited to short-lived jobs.

**Prometheus**

6. **Prometheus architecture?** Server (scrape, TSDB, PromQL, rules) + exporters/client libraries + service discovery + Alertmanager + Pushgateway + remote storage; Grafana for dashboards.
7. **Data model?** Time series identified by metric name + labels; samples are float64 values with millisecond timestamps.
8. **Metric types?** Counter, gauge, histogram, summary.
9. **What are `job` and `instance`?** Group of similar targets; a single `host:port` target.
10. **What is Node Exporter and its port?** Exposes Linux host metrics at `/metrics` on 9100.
11. **What is cardinality and why dangerous?** Number of unique series; unbounded labels explode memory.
12. **`relabel_configs` vs `metric_relabel_configs`?** Before scrape on target labels vs after scrape on sample labels.
13. **Default retention and sizing formula?** 15 days; disk = retention × samples/s × ~1-2 bytes.

**PromQL**

14. **`rate` vs `irate` vs `increase`?** Per-second average over window; per-second from last two samples; total increase in the window.
15. **CPU usage % query?** `100 - avg by (instance)(rate(node_cpu_seconds_total{mode="idle"}[5m]))*100`.
16. **Why MemAvailable?** It includes reclaimable cache; MemFree understates usable memory.
17. **p95 latency query?** `histogram_quantile(0.95, sum by (le)(rate(..._bucket[5m])))`.
18. **Why does a binary operation return empty?** Label sets differ; use `on(...)`/`ignoring(...)`.
19. **What is a recording rule?** Precomputed query stored as a new series for speed/simplicity.

**Alerting**

20. **Alert rule fields and states?** `alert, expr, for, labels, annotations`; inactive → pending → firing → resolved.
21. **What does Alertmanager do?** Dedupe, group, route, inhibit, silence, notify (with HA clustering).
22. **`group_wait`, `group_interval`, `repeat_interval`?** Initial delay, delay for new alerts in a group, reminder frequency.
23. **Inhibition vs silence?** Automatic suppression by another alert vs manual time-limited mute.
24. **How did you validate alert generation in the lab?** Stressed CPU/memory/disk/stopped the exporter and watched pending/firing in Prometheus, Alertmanager UI and webhook output, then resolved.
25. **What is burn-rate alerting?** Alerting on error-budget consumption speed with multiple windows (e.g., 14.4× over 1h and 5m).

**Grafana, scaling, advanced**

26. **Grafana data source and variables?** Connection to Prometheus; dropdowns from `label_values()` used with `=~` to reuse dashboards.
27. **Dashboard design principles?** One purpose, key stats first, USE/RED layout, correct units/thresholds, drill-downs, variables, documentation, dashboards as code.
28. **How to scale Prometheus?** Optimize cardinality, HA replicas, shard by function/hashmod, federate aggregates, remote write to Thanos/Mimir/VictoriaMetrics, managed services.
29. **How do you monitor Kubernetes?** kube-prometheus-stack: node-exporter, cAdvisor, kube-state-metrics, ServiceMonitors/PodMonitors and PrometheusRules via the Operator.
30. **Advanced observability strategy?** OpenTelemetry instrumentation + Collector, structured logs (Loki), traces (Tempo/Jaeger), correlated by trace IDs/exemplars, SLO-based alerts, synthetic/RUM, canary analysis and chaos engineering with governance of cost/privacy.

---

# 1️⃣1️⃣ COMMON UNIT IV TRAPS 🚨

```
1. Using rate() on a gauge, or graphing a raw counter
2. Using the wrong rate window (smaller than 2-4 scrape intervals)
3. Averaging percentiles (avg of p95); forgetting `le` in histogram aggregations
4. Binary operations with mismatched labels (e.g., node_load1 / count by (instance)(...) → EMPTY): use on(instance)
5. Using MemFree as "free memory"; confusing percent (0-100) with ratio (0-1) units in Grafana
6. Alerting on every cause (CPU > 80%) instead of symptoms/SLOs; no `for:`; no runbook; no owner
7. Thinking Alertmanager evaluates alerts (Prometheus does; Alertmanager routes/notifies)
8. Putting a load balancer in front of clustered Alertmanagers; HA pair without a replica label
9. Unbounded labels (user_id, URL, pod UID) → cardinality explosion
10. Using federation to copy all data; running Pushgateway for normal services
11. Leaving Prometheus/Alertmanager/Grafana/Node Exporter ports open to the internet
12. Using = instead of =~ with multi-value Grafana variables
13. Believing monitoring = observability, or that three tools = three pillars working together (correlation is the point)
14. Averages instead of percentiles for latency; ignoring tail latency
15. Treating dashboards as a substitute for alerts (eyeball monitoring)
16. Not monitoring the monitoring system (no Watchdog/meta-monitoring)
17. Forgetting to clean up stress files/instances after the lab
18. Quoting exact versions/pricing/product availability as permanent facts (they change: say "currently" or "check the docs")
```

---

# 1️⃣2️⃣ WHAT TO WRITE IN AN EXAM ✍️

| Question type | Structure |
|---|---|
| **Monitoring vs observability / pillars** | Definitions + comparison table + three pillars with example each + correlation + Golden Signals/RED/USE |
| **Prometheus architecture** | Labeled diagram (targets/exporters → scrape → TSDB → PromQL → Grafana; rules → Alertmanager) + component roles + metric types + pull model |
| **PromQL** | Data types, selectors, `rate`, aggregation, histogram quantile, 4-5 real queries (CPU/memory/disk/network/error rate) |
| **Alerting** | Rule anatomy + lifecycle + Alertmanager features (grouping/routing/inhibition/silence) + config snippet + best practices |
| **Grafana dashboards** | Data source, panels, variables, thresholds/units, design principles, example panels for CPU/memory/storage/network |
| **Scaling monitoring** | Capacity formula + ladder (optimize → HA → sharding → federation → remote write → Thanos/Mimir) + Kubernetes + managed + cost |
| **Advanced observability** | Strategy + OpenTelemetry + logs/traces + correlation + SLO alerting + synthetic/RUM + AIOps/chaos + governance |
| **Practical 8/9** | Aim, requirements, architecture diagram, numbered steps with commands/configs, outputs, alert validation table, dashboard screenshots, trend analysis, cleanup, result |

Writing habits: underline keywords (scrape, label, `rate()`, `for`, group_by, inhibition, `$__rate_interval`, cardinality, remote write), draw the architecture diagram, give one query and one config snippet, mention one tool per concept.

---

# 1️⃣3️⃣ WHAT NOT TO WRITE ❌

```
✘ "Prometheus pushes metrics" (it pulls; Pushgateway is the exception for batch jobs)
✘ "Grafana stores the metrics" (it queries data sources)
✘ "Alertmanager creates alerts" (Prometheus evaluates rules; Alertmanager manages notifications)
✘ Averages for latency without percentiles ; PromQL without rate() for counters
✘ Exact prices, version numbers or "latest" claims stated as permanent
✘ Alerts on every metric / dashboards with 60 panels as "best practice"
✘ Insecure sample setups (open 9090/9093/3000 to the world, default admin password) presented as recommended
✘ Mixing up SLI/SLO/SLA or monitoring/observability definitions
```

---

# 1️⃣4️⃣ UNIT IV MASTER SHEET 🧠🔥

```
THEORY     monitoring (known) ⊂ observability (unknown) ; pillars metrics/logs/traces (+ events, profiles) ; correlation ; Golden Signals / RED / USE ; SLI→SLO→SLA ; error budget
PROMETHEUS pull + TSDB + PromQL + rules ; scrape_interval 15s ; labels job/instance ; types counter/gauge/histogram/summary ; exporters (node :9100) ; Pushgateway only for batch ;
           cardinality ; relabeling ; service discovery ; retention 15d ; promtool ; instrument with client libs
PROMQL     rate (counters) ; by/without ; histogram_quantile(sum by le of rate buckets) ; predict_linear ; on()/ignoring() ; recording rules ; window ≥ 4× interval
ALERTING   rule (expr, for, labels, annotations) → pending → firing → Alertmanager (group, route, inhibit, silence, notify, HA) ; webhook/email/Slack/PagerDuty ; burn-rate alerts ; Watchdog
GRAFANA    data source → panels → variables (=~) → thresholds/units → annotations → provisioning ; USE/RED dashboards ; import 1860 ; Grafana alerting
SCALING    optimize → HA → shard → federate (aggregates) → remote_write/Agent → Thanos/Mimir/VM/managed ; capacity formulas ; Kubernetes Operator + ServiceMonitor ; cost + meta-monitoring
ADVANCED   OTel (SDK + Collector + OTLP) ; structured logs + Loki/LogQL ; traces (W3C traceparent, sampling) ; exemplars ; SLO-based alerts ; synthetic/RUM ; profiling ; eBPF ; AIOps ; chaos
LABS       P8: Node Exporter + Prometheus + Alertmanager + rules + stress tests + webhook proof ; P9: Grafana + data source + dashboards + load + trend analysis ; ALWAYS clean up
RULES      symptoms not causes ; percentiles not averages ; low-cardinality labels ; protect ports ; monitor the monitors ; dashboards/rules in Git
```

---

# 1️⃣5️⃣ MEMORY MAPS 🗺️

## Stack map
```
Targets (apps, Node Exporter, DB exporters) ─/metrics─▶ PROMETHEUS ──rules──▶ ALERTMANAGER ─▶ Email / Slack / PagerDuty / Webhook
                                                │  PromQL
                                                ▼
                                            GRAFANA (dashboards, explore, alerts) ◀── Loki (logs), Tempo (traces)
                                                ▲ remote_write ─▶ Thanos / Mimir / managed (long-term, global)
```

## Alert flow
```
metric → expr true → PENDING (for) → FIRING → Alertmanager: group → route → (inhibit/silence?) → receiver → RESOLVED
```

## Observability flow
```
Alert (SLO burn) → dashboard (RED) → trace (where) → logs (why) → profile (which function) → mitigate (rollback/flag) → postmortem → improve
```

## Scaling ladder
```
Optimize → HA pair → functional/hashmod shards → federation (aggregates) → remote_write → Thanos / Mimir / VictoriaMetrics / managed → (cost + meta-monitoring)
```

---

# 1️⃣6️⃣ HOW UNIT IV CONNECTS TO THE REST OF INT378 🔗

| Unit | Connection |
|---|---|
| **Unit I** Cloud and Docker | Monitor EC2/containers; CloudWatch vs Prometheus; autoscaling driven by metrics; Netflix/Uber observability and Chaos Monkey; cAdvisor/Docker metrics |
| **Unit II** IaC | Provision Prometheus/Grafana/alerts with Terraform/CloudFormation; drift detection; dashboards and rules as code; `promtool`/`amtool` in CI |
| **Unit III** Agile and CI/CD | Post-deploy verification, canary analysis, deployment annotations, DORA/Jenkins metrics, feedback loop of the DevOps cycle, error budgets governing release speed |
| **Unit V** DevSecOps and cloud security | Security monitoring (audit logs, Falco, SIEM), securing Prometheus/Grafana/Alertmanager, secrets handling, anomaly detection, IAM for telemetry |
| **Unit VI** Cloud-native and cost | SLO governance, telemetry cost control, cost dashboards/alarms, 12-factor logging to stdout, serverless/cloud metrics |

```
Unit I (what runs) → Unit II (define as code) → Unit III (deliver continuously) → Unit IV (observe) → Unit V (secure) → Unit VI (optimize and govern)
```

---

# 🏆 UNIT IV — FINAL TAKEAWAY

```
1. Monitoring answers "is it broken?"; observability answers "why?" using correlated metrics, logs and traces.
2. Measure the right things: Golden Signals/RED for services, USE for resources, SLOs and error budgets for user impact.
3. Prometheus collects (pull), stores (TSDB with labels) and queries (PromQL) metrics; exporters and client libraries produce them.
4. Alert on symptoms: rules decide when, Alertmanager decides who and how (group, route, inhibit, silence).
5. Grafana turns data into performance dashboards: data sources, panels, variables, thresholds, drill-down, dashboards as code.
6. Scale deliberately: optimize cardinality, add HA, shard, federate aggregates, store long-term with Thanos/Mimir or managed services; control cost; monitor the monitors.
7. Go deeper with OpenTelemetry, Loki, Tempo, SLO burn-rate alerting, synthetic/RUM, profiling, eBPF, chaos engineering and governance.
8. You can do it yourself: Node Exporter + Prometheus + Alertmanager with validated alerts, and Grafana dashboards with trend analysis.
```

---

## 🫡 UNIT IV STATUS

✅ Part 1 Monitoring and observability from scratch
✅ Part 2 Prometheus fundamentals
✅ Part 3 PromQL deep dive
✅ Part 4 Alerting (Prometheus rules + Alertmanager)
✅ Part 5 Grafana and performance dashboards
✅ Part 6 Scaling monitoring solutions
✅ Part 7 Advanced observability strategies
✅ Part 8 Practical lab guide (Prometheus, Node Exporter, alerts, Grafana)
✅ Grand Consolidation
✅ Lab bundle: `U4_lab_files.zip` and `U4_infra-dashboard.json`

## Next up in your syllabus: UNIT V, DevSecOps and Cloud Security (securing CI/CD pipelines, IAM, Kubernetes and container security, secrets and key management, emerging trends in DevOps security) 🚀

Send the go-ahead and I'll build Unit V the same way.
