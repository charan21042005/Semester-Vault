# 🟢 UNIT IV — PART 1: MONITORING AND OBSERVABILITY FROM ABSOLUTE SCRATCH

> **Covered in this document:** Unit IV roadmap → why systems need eyes (the cost of not knowing) → what **monitoring** is → what **observability** is (origin, known-unknowns vs unknown-unknowns) → **monitoring vs observability** table → the **three pillars** (metrics, logs, traces) plus events and profiles → telemetry pipeline (instrument → collect → store → query → visualize → alert → act) → kinds of monitoring (infrastructure, application/APM, network, synthetic, RUM, log, security, business, container/Kubernetes, cloud) → **what to measure**: Four Golden Signals, RED method, USE method → **SLI, SLO, SLA and error budgets** with the "nines" table → white-box vs black-box, symptom vs cause → push vs pull collection → alerting philosophy, alert fatigue, incident metrics (MTTD, MTTA, MTTR, MTBF) → monitoring inside the DevOps loop (link to Units I-III) → the tool landscape (open-source vs SaaS vs cloud-native) → observability maturity ladder → an incident dry run → common mistakes → viva → master sheet.

---

Absolutely mate. 😤🔥 Unit IV starts now.

Same exact system as Units I, II and III:

> Absolute scratch → intuition → concept → diagram → commands/code → dry run → practical thinking → exam/viva → STOP.

Your Unit IV syllabus is:

> **Monitoring and Observability:** Monitoring and observability with tools like Prometheus and Grafana, data collection, metrics, alerting, and the creation of performance dashboards, Scaling monitoring solutions, and implementing advanced observability strategies.

Practicals attached to this unit:

```
Practical 8 → Install and configure Prometheus, deploy Node Exporter, collect CPU, memory, disk and network metrics,
              define alerting rules, and validate alert generation for infrastructure monitoring
Practical 9 → Deploy Grafana, integrate it with Prometheus as a data source, design monitoring dashboards for infrastructure
              performance, visualize CPU, memory, storage and network utilization, and analyze system performance trends
```

(The practical list numbers these 8 and 9 in your syllabus: Practical 6 was Jenkins in Unit III, Practical 7 is DevSecOps in Unit V.)

---

# 🚀 UNIT IV — ROADMAP

```
PART 1 — Monitoring and Observability from Absolute Scratch ⭐          (this file)
PART 2 — Prometheus Fundamentals (architecture, data model, metric types, scraping, exporters, instrumentation) 🔥
PART 3 — PromQL Deep Dive (querying: rate, aggregation, histograms, recording rules, node-exporter queries) 🧮
PART 4 — Alerting (alert rules, Alertmanager routing/grouping/inhibition/silencing, receivers, SLO burn-rate alerts) 🚨
PART 5 — Grafana and Performance Dashboards (data sources, panels, variables, design, provisioning as code) 📊
PART 6 — Scaling Monitoring Solutions (federation, sharding, remote write, Thanos/Mimir/VictoriaMetrics, cardinality, Kubernetes, cloud) 📈
PART 7 — Advanced Observability Strategies (logs, traces, OpenTelemetry, SLO-based alerting, synthetic, eBPF, AIOps, chaos) 🔭
PART 8 — Practical Lab Guide (Prometheus + Node Exporter + alert rules, and Grafana dashboards: step by step) 🧪

🏆 GRAND CONSOLIDATION — revision, comparison tables, 30 viva Qs, traps, master sheet
```

---

# 1️⃣ WHY SYSTEMS NEED EYES 👀

## 1.1 The story so far

```
Unit I   → you built infrastructure and containers
Unit II  → you defined it as code
Unit III → a pipeline now ships changes many times a day
Unit IV  → ...but once it is running in production, HOW DO YOU KNOW IT IS HEALTHY?
```

A fast delivery pipeline without visibility is like driving a fast car with the dashboard covered. Fast + blind = crash.

## 1.2 What happens without monitoring

| Situation | Consequence |
|---|---|
| Users find outages before you do | Lost customers, bad reviews ("Twitter is your monitoring system") |
| Disk fills silently at 3 AM | Database crashes; recovery takes hours |
| Memory leak grows over days | Random restarts, nobody knows why |
| Slow response, no data | Teams argue from opinions, not evidence |
| Capacity planning by guesswork | Over-spend or sudden overload on sale day |
| No history | "It was fine last week?" cannot be answered |

Cars analogy 🚗: the dashboard shows speed, fuel, engine temperature and warning lights. Modern cars add black-box recorders and diagnostics. **Monitoring = dashboard + warning lights. Observability = being able to diagnose *any* problem from the recorded data, even one you never expected.**

## 1.3 Goals of monitoring and observability

```
1. DETECT problems fast (before users complain)          4. UNDERSTAND system behavior and trends
2. DIAGNOSE root causes quickly (reduce MTTR)            5. PLAN capacity and optimize cost
3. ALERT the right people with actionable information    6. PROVE reliability to the business (SLOs/SLAs) and learn from incidents
```

---

# 2️⃣ WHAT IS MONITORING? ⭐

> **Monitoring** is the practice of **collecting, storing, visualizing and alerting on predefined measurements (metrics, checks, logs)** about systems so you know whether they are working and can respond when they are not.

Monitoring answers **known questions**: *Is the CPU above 80%? Is the website up? Is the disk almost full? Is the error rate above 1%?*

```
You decide in advance WHAT to measure and WHAT is "bad".
The tool watches those things and shouts when they cross a line.
```

## 2.1 What monitoring does

| Function | Example |
|---|---|
| **Check** | Is the service responding on port 443? |
| **Measure** | CPU 62%, 450 requests/second |
| **Record history** | Keep 15 days of CPU data to see trends |
| **Visualize** | Dashboards (Grafana) |
| **Alert** | Email/Slack/PagerDuty when disk > 90% |

## 2.2 Limits of monitoring

Monitoring works great for **known failure modes**. Modern systems (microservices, containers, serverless) fail in **new, surprising ways**: a combination of a slow dependency + a retry storm + a specific customer's data. You can't pre-write a dashboard or alert for every combination. That's where observability enters.

---

# 3️⃣ WHAT IS OBSERVABILITY? ⭐⭐⭐

## 3.1 Definition

> **Observability** is the ability to understand a system's **internal state** by examining the **data it produces on the outside** (telemetry), so you can ask **new questions** and find the cause of problems you did not anticipate, **without shipping new code**.

Origin: from **control theory** (Rudolf Kálmán, 1960): a system is "observable" if its internal state can be inferred from its outputs.

```
Monitoring    → "Is something wrong?"                       (known unknowns: you know what to look for)
Observability → "WHY is it wrong, and where exactly?"       (unknown unknowns: you didn't know to look for it)
```

Doctor analogy 🩺: monitoring = checking temperature and pulse (known vital signs). Observability = having full test results, scans and history so a doctor can diagnose a rare condition nobody predicted.

## 3.2 Known-unknowns vs unknown-unknowns

| | Known-unknowns | Unknown-unknowns |
|---|---|---|
| Meaning | You know the question | You don't know what to ask |
| Example | "Is CPU > 90%?" | "Why are only users in Pune on mobile Safari timing out at checkout since the 14:05 deploy?" |
| Approach | Dashboards + threshold alerts | Rich, high-cardinality telemetry + ad-hoc exploration |
| Practice | Monitoring | Observability |

## 3.3 Monitoring vs Observability ⭐⭐⭐ (WRITE THIS TABLE)

| Aspect | Monitoring | Observability |
|---|---|---|
| Core question | Is it working? | Why isn't it working? |
| Approach | Predefined metrics, thresholds, dashboards | Explore rich telemetry to ask arbitrary questions |
| Data | Mostly metrics and checks | Metrics + logs + traces (+ events, profiles), correlated |
| Failure types | Known, anticipated | Unknown, novel, complex |
| Output | Alerts and dashboards | Understanding and root cause |
| Mindset | Reactive: "tell me when X breaks" | Proactive/investigative: "let me find out what's happening" |
| Relationship | **A subset/part of observability** | Superset: includes monitoring |
| Typical tools | Prometheus alerts, Nagios, Zabbix | OpenTelemetry, Grafana stack, Datadog, Honeycomb |
| Instrumentation | Basic (host/app metrics) | Deep, structured, context-rich |

```
Observability ⊃ Monitoring
You need BOTH: monitoring (alerts when something breaks) + observability (tools to find out why).
```

---

# 4️⃣ THE THREE PILLARS OF OBSERVABILITY ⭐⭐⭐

```
                    OBSERVABILITY
        ┌───────────────┼───────────────┐
     METRICS           LOGS           TRACES
   (what/how much)  (what happened)  (where/how long across services)
```

## 4.1 Metrics 📈

> **Metrics** are **numeric measurements sampled over time** (time series), cheap to store and great for trends, dashboards and alerts.

Examples: CPU utilization 0.62, `http_requests_total` = 18,240, memory used = 3.1 GB, queue length = 14, request latency p95 = 240 ms.

| Strengths | Weaknesses |
|---|---|
| Compact, cheap, fast to query | Lose detail (aggregates hide individual requests) |
| Perfect for alerting and trends | "Why?" needs more data |
| Predictable storage cost (if cardinality is controlled) | High-cardinality labels explode cost (Part 6) |

Tools: **Prometheus**, CloudWatch metrics, StatsD, InfluxDB.

## 4.2 Logs 📜

> **Logs** are **timestamped records of discrete events** emitted by software ("user 42 failed login", stack traces).

```
2026-11-23T10:15:42.123Z level=ERROR service=checkout order_id=9181 user=42 msg="payment gateway timeout" duration_ms=5000
```

| Strengths | Weaknesses |
|---|---|
| Rich detail and context; great for debugging specific events | Large volume = expensive; slow to search at scale |
| Human-readable; audit trail | Unstructured text hard to query (use **structured JSON logs**) |

Tools: **ELK/EFK (Elasticsearch, Logstash/Fluentd, Kibana)**, **Grafana Loki**, Splunk, CloudWatch Logs, OpenSearch.

## 4.3 Traces 🧵

> A **distributed trace** follows **one request across many services**, recording each step as a **span** (with start time, duration, metadata) so you see **where time was spent** and **which service failed**.

```
Trace ID: 7f3a...     (one user request: GET /checkout)
 └─ span: api-gateway          [0 ─────────────────────────── 820 ms]
     └─ span: checkout-service      [20 ───────────────────── 800 ms]
         ├─ span: cart-service          [25 ── 60 ms]
         ├─ span: payment-service       [70 ──────────────── 790 ms]   ← slow!
         │    └─ span: bank-API-call        [90 ────────────── 780 ms]   ← root cause: external dependency
         └─ span: inventory-service     [30 ── 80 ms]
```

| Strengths | Weaknesses |
|---|---|
| Shows request path and latency breakdown in microservices | Needs instrumentation in every service |
| Finds bottlenecks and failing dependencies | Data volume large → **sampling** needed |

Tools: **OpenTelemetry (standard)**, **Jaeger**, **Grafana Tempo**, Zipkin, AWS X-Ray.

## 4.4 How the three pillars work together ⭐

```
METRICS  tell you SOMETHING is wrong:    "p95 latency for /checkout rose from 200 ms to 2 s at 14:05"   (alert fires)
TRACES   tell you WHERE:                  "the payment-service → bank-API span takes 1.8 s"
LOGS     tell you WHY:                    "bank-API returned 503 'rate limit exceeded' for client_id=XYZ"
```

The key capability is **correlation**: jump from a metric spike → to traces from the same time → to logs of the same trace ID. Modern stacks add **exemplars** (trace IDs attached to metrics) and shared labels (Part 7).

## 4.5 Beyond the three pillars

| Signal | Meaning |
|---|---|
| **Events** | Discrete important happenings: deployments, config changes, feature-flag flips, scaling actions (shown as annotations on dashboards) |
| **Profiles (continuous profiling)** | Which functions burn CPU/memory over time (Pyroscope, Parca, Grafana Pyroscope); sometimes called the 4th pillar |
| **Synthetic checks** | Scripted probes that act like users |
| **Real-user monitoring (RUM)** | Browser/mobile performance from actual users |
| **Metadata/topology** | Service maps, dependencies |

## 4.6 Comparison of the pillars

| | Metrics | Logs | Traces |
|---|---|---|---|
| Data type | Numbers over time | Text/JSON events | Request journeys (spans) |
| Granularity | Aggregated | Per event | Per request |
| Storage cost | Low | High | Medium-high (sampled) |
| Best for | Alerting, trends, capacity | Debugging details, audit | Latency, dependencies |
| Query | PromQL | LogQL, Lucene/KQL | Trace search, TraceQL |
| Example tool | Prometheus | Loki/ELK | Jaeger/Tempo |

---

# 5️⃣ THE TELEMETRY PIPELINE 🔄

```
 INSTRUMENT  →  COLLECT  →  TRANSPORT/PROCESS  →  STORE  →  QUERY  →  VISUALIZE  →  ALERT  →  ACT
  (apps,        (agents,     (filter, enrich,       (TSDB,    (PromQL,  (Grafana    (Alertmanager, (on-call,
   exporters)    scrapers)    sample, route)         log/trace  LogQL)    dashboards)  PagerDuty)    runbook,
                                                     stores)                                         auto-remediation)
```

| Stage | What happens | Examples |
|---|---|---|
| **Instrument** | Code/hosts emit telemetry | Client libraries, exporters, OpenTelemetry SDK, structured logging |
| **Collect** | Gather from sources | Prometheus scrape, Fluent Bit, OTel Collector, CloudWatch agent |
| **Process/transport** | Filter, enrich, batch, sample, route | OTel Collector, Logstash, Vector, relabeling |
| **Store** | Time-series DB / log store / trace store | Prometheus TSDB, Loki, Tempo, Elasticsearch |
| **Query** | Ask questions | PromQL, LogQL, SQL, TraceQL |
| **Visualize** | Dashboards | Grafana, Kibana |
| **Alert** | Evaluate rules, notify | Prometheus rules + Alertmanager, Grafana Alerting |
| **Act** | Humans/automation respond | Runbooks, auto-scaling, rollback, incident management |

---

# 6️⃣ KINDS OF MONITORING 🧭

| Type | Watches | Examples of metrics | Tools |
|---|---|---|---|
| **Infrastructure/host** | Servers, VMs, disks, network | CPU, memory, disk, network I/O | Node Exporter, CloudWatch, Zabbix |
| **Application (APM)** | Code behavior | Request rate, errors, latency, DB query time | OpenTelemetry, Datadog APM, New Relic, Dynatrace |
| **Container/Kubernetes** | Pods, nodes, cluster | Pod CPU/memory, restarts, pending pods | cAdvisor, kube-state-metrics, kube-prometheus-stack |
| **Network** | Connectivity, bandwidth | Latency, packet loss, throughput | SNMP, Blackbox exporter, ping/traceroute |
| **Log monitoring** | Log streams | Error counts, patterns | ELK, Loki, Splunk |
| **Synthetic monitoring** | Availability from outside via scripted probes | Uptime, response time, SSL expiry | Blackbox exporter, Pingdom, Checkly, CloudWatch Synthetics |
| **Real User Monitoring (RUM)** | Real users' browsers/apps | Page load, Core Web Vitals | Grafana Faro, Datadog RUM, Google Analytics |
| **Database** | DB engines | Connections, slow queries, replication lag | mysqld_exporter, postgres_exporter, mongodb_exporter (Unit V Mongo link 🔗) |
| **Security monitoring (SIEM)** | Threats and audit | Failed logins, anomalies | Splunk, Wazuh, GuardDuty (Unit V) |
| **Business/product** | Outcomes | Orders/min, sign-ups, revenue | Custom metrics + dashboards |
| **Cloud/cost** | Cloud services and spend | Service health, budget burn | CloudWatch, Azure Monitor, Cost Explorer |
| **Pipeline/CI-CD** | Delivery performance | Build duration, failure rate, DORA | Jenkins Prometheus plugin (Unit III link) |

---

# 7️⃣ WHAT TO MEASURE: GOLDEN SIGNALS, RED AND USE ⭐⭐⭐

Too many metrics = noise. Three well-known frameworks tell you **which** to watch first.

## 7.1 The Four Golden Signals (Google SRE book) 🥇

| Signal | Meaning | Example |
|---|---|---|
| **Latency** | Time to serve a request (track **successful vs failed** separately; use percentiles) | p50 = 80 ms, p99 = 900 ms |
| **Traffic** | Demand on the system | 1,200 requests/second |
| **Errors** | Rate of failing requests (explicit 5xx, wrong content, or too slow) | 0.4% 5xx |
| **Saturation** | How "full" the system is (the most constrained resource) | Memory 92% used, queue depth growing |

Memory trick: **L-T-E-S**.

## 7.2 RED method (for request-driven services, by Tom Wilkie) 🔴

| Letter | Meaning | PromQL idea (Part 3) |
|---|---|---|
| **R**ate | Requests per second | `rate(http_requests_total[5m])` |
| **E**rrors | Failed requests per second (or ratio) | `rate(http_requests_total{status=~"5.."}[5m])` |
| **D**uration | Latency distribution | `histogram_quantile(0.95, ...)` |

Use RED for **microservices/APIs** (what users experience).

## 7.3 USE method (for resources, by Brendan Gregg) 🔵

For every **resource** (CPU, memory, disk, network interface): check **U**tilization, **S**aturation, **E**rrors.

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| **CPU** | % busy | Run-queue length / load average > cores | Machine-check/hardware errors |
| **Memory** | % used | Swapping, page scans, OOM events | Allocation failures, OOM kills |
| **Disk** | % full; I/O busy time | I/O queue length / wait time | I/O errors |
| **Network** | Bandwidth used vs capacity | Dropped/queued packets, retransmits | Interface errors, drops |

Use USE for **infrastructure** (your Practicals 8 and 9 are USE-style dashboards).

## 7.4 Which method when?

```
RED  → "Is my SERVICE healthy for users?"       (rate, errors, duration)
USE  → "Is my RESOURCE healthy?"                (utilization, saturation, errors)
Four Golden Signals ≈ RED + Saturation (general-purpose, for any user-facing system)
```

## 7.5 What to measure at each layer (checklist) ✅

```
User/business:  conversion, orders/min, sign-ups, error pages seen by users, Core Web Vitals
Application:    RED metrics per endpoint, queue depth, cache hit ratio, dependency latency, thread/connection pool use
Platform:       container CPU/memory/restarts, pod status, autoscaler events, load balancer 5xx and healthy hosts
Infrastructure: CPU, memory, disk space + I/O, network, system load, file descriptors, clock sync
Delivery:       deployment frequency, change failure rate, build time (DORA)
```

---

# 8️⃣ SLI, SLO, SLA AND ERROR BUDGETS ⭐⭐⭐ (SRE VOCABULARY)

| Term | Full form | Meaning | Example |
|---|---|---|---|
| **SLI** | Service Level **Indicator** | A **measured** metric of service quality | % of requests returning non-5xx; % of requests faster than 300 ms |
| **SLO** | Service Level **Objective** | The **target** for an SLI over a window (internal goal) | 99.9% of requests succeed over 30 days |
| **SLA** | Service Level **Agreement** | A **contract** with customers, with consequences (credits/penalties) if missed | "99.5% monthly uptime or 10% credit" (SLA is usually looser than the SLO) |
| **Error budget** | 100% − SLO | The **allowed** unreliability; spend it on risk, releases, experiments | 0.1% of requests may fail in 30 days |

```
SLI (what we measure)  →  SLO (what we promise ourselves)  →  SLA (what we promise customers, with penalties)
SLA is looser than SLO so that you get warning before you breach the contract.
```

## 8.1 The "nines" table ⭐

| Availability | Downtime per year | Downtime per month (30.44 d) | Per week |
|---|---|---|---|
| 99% ("two nines") | 3.65 days | 7.31 hours | 1.68 hours |
| 99.9% ("three nines") | 8.77 hours | 43.8 minutes | 10.1 minutes |
| 99.95% | 4.38 hours | 21.9 minutes | 5.04 minutes |
| 99.99% ("four nines") | 52.6 minutes | 4.38 minutes | 1.01 minutes |
| 99.999% ("five nines") | 5.26 minutes | 26.3 seconds | 6.05 seconds |

Each extra nine costs far more engineering effort and money; choose the SLO that users actually need.

## 8.2 Error budget in practice ✅

```
SLO: 99.9% of requests succeed over 30 days  →  error budget = 0.1%
Traffic: 10,000,000 requests/month → allowed failures = 10,000 requests
Mid-month: 6,000 failed → 60% of budget burned
Policy:
  Budget healthy → ship features, run risky experiments, deploy faster
  Budget nearly spent → freeze risky releases, prioritize reliability work
```

This turns "reliability vs features" arguments into **data-driven agreements** between developers and operations (SRE culture; links to Unit III's DORA and release practices).

## 8.3 Good SLIs

| Service type | Typical SLIs |
|---|---|
| Request/response APIs | Availability (success ratio), latency (% under threshold) |
| Data pipelines | Freshness, completeness, correctness |
| Storage | Durability, availability |
| Batch jobs | Success rate, completion time |

Use **percentiles (p95/p99)** for latency, not averages: averages hide pain.

---

# 9️⃣ MONITORING STYLES: BLACK-BOX vs WHITE-BOX, SYMPTOMS vs CAUSES, PUSH vs PULL ⭐

## 9.1 Black-box vs white-box

| | **Black-box monitoring** | **White-box monitoring** |
|---|---|---|
| Looks at | System from the **outside** (like a user) | System **internals** (instrumented metrics, logs) |
| Examples | HTTP probe: "does the homepage return 200 in < 2 s?"; ping; SSL expiry check | `http_requests_total`, thread pool use, GC pauses |
| Detects | Current, user-visible failures (**symptoms**) | Causes, impending problems, internals |
| Tool | **Blackbox exporter**, synthetic checks | Prometheus client libraries, exporters |

Use both: black-box for "is it really up for users?", white-box for "why?".

## 9.2 Symptoms vs causes (alerting principle) ⭐

```
SYMPTOM: "users see errors / pages are slow"      ← page a human for this (it matters now)
CAUSE:   "CPU 85%, a replica is down"              ← usually a ticket/dashboard, not a 3 AM page, unless it is certain to hurt users
```

## 9.3 Push vs pull collection ⭐⭐

| | **Pull** (Prometheus) | **Push** (StatsD, CloudWatch agent, OpenTelemetry exporters, Pushgateway) |
|---|---|---|
| Who initiates | The monitoring server **scrapes** targets over HTTP | Targets **send** data to the collector |
| Target discovery | Server must know targets (service discovery) | Targets just need the collector's address |
| Health insight | A failed scrape = target is down (the `up` metric) | Silence is ambiguous (is it down or just quiet?) |
| Firewalls/NAT | Server must reach targets | Works from behind NAT; good for short-lived jobs |
| Control | Central control of scrape rate/targets | Senders control volume |
| Best for | Long-running services, infrastructure | Short-lived batch jobs, edge devices, serverless |

Prometheus is **pull-based** (with a Pushgateway for short-lived jobs); many cloud tools are push-based. Neither is "better": they fit different situations.

---

# 🔟 ALERTING PHILOSOPHY AND INCIDENT METRICS 🚨

## 10.1 Good alerts are...

```
✔ ACTIONABLE        (a human knows what to do; link a runbook)
✔ URGENT            (needs attention now; otherwise it's a ticket or a dashboard)
✔ SYMPTOM-BASED     (user impact, SLO burn) rather than every low-level twitch
✔ CLEAR             (what's broken, where, since when, how bad, which dashboard)
✔ OWNED             (routes to the right team)
```

## 10.2 Alert fatigue ⚠️

> **Alert fatigue**: so many (often false or low-value) alerts that on-call engineers start ignoring them, and a real incident gets missed.

Causes: thresholds too sensitive, no `for:` delay (flapping), alerting on causes not symptoms, duplicate alerts, no ownership. Fixes: tune thresholds, use `for` durations, group and deduplicate, inhibit child alerts, review noisy alerts weekly (Part 4).

## 10.3 Severity levels (typical)

| Severity | Meaning | Response |
|---|---|---|
| **P1/critical** | Major user impact, outage | Page immediately, 24×7 |
| **P2/high** | Partial impact or imminent risk | Page in business hours/short SLA |
| **P3/warning** | Degradation, needs attention soon | Ticket/Slack |
| **P4/info** | FYI | Dashboard/log |

## 10.4 Incident metrics ⭐

| Metric | Meaning |
|---|---|
| **MTTD** | Mean Time To Detect: from failure start to detection |
| **MTTA** | Mean Time To Acknowledge: alert to human ack |
| **MTTR** | Mean Time To Recover/Resolve (also read as Respond/Repair; state which you mean): detection to restoration |
| **MTBF** | Mean Time Between Failures: reliability |
| **MTTF** | Mean Time To Failure (non-repairable) |

```
Availability ≈ MTBF / (MTBF + MTTR)    → improve availability by failing less (↑MTBF) AND recovering faster (↓MTTR)
Observability mostly attacks MTTD and MTTR.
```

---

# 1️⃣1️⃣ MONITORING IN THE DEVOPS LOOP 🔗

```
PLAN → CODE → BUILD → TEST → RELEASE → DEPLOY → OPERATE → MONITOR ──┐
  ▲                                                                   │
  └────────────── feedback: metrics, errors, user behavior ◀──────────┘
```

| Unit | Link to monitoring |
|---|---|
| **Unit I** | Monitor EC2/containers; autoscaling uses metrics (CloudWatch → scale out); Netflix/Uber observability |
| **Unit II** | Provision Prometheus/Grafana/alarms with Terraform/CloudFormation; detect drift |
| **Unit III** | Post-deploy verification, canary analysis, feature-flag metrics, A/B guardrails, DORA metrics, Jenkins metrics |
| **Unit V** | Security monitoring, audit logs, anomaly detection, secrets and access logs |
| **Unit VI** | Cost monitoring, billing alarms, SLO governance |

Monitoring is how DevOps closes the feedback loop: deploy → observe → learn → improve.

---

# 1️⃣2️⃣ THE TOOL LANDSCAPE 🧰

| Category | Open source / self-managed | SaaS / commercial | Cloud-native |
|---|---|---|---|
| **Metrics + alerting** | **Prometheus**, VictoriaMetrics, Thanos/Mimir (scale), Nagios, Zabbix, Icinga, InfluxDB+Telegraf | Datadog, New Relic, Dynatrace, Grafana Cloud | **AWS CloudWatch**, Azure Monitor, Google Cloud Monitoring, Amazon Managed Service for Prometheus |
| **Visualization** | **Grafana**, Kibana | Datadog dashboards, Grafana Cloud | CloudWatch dashboards, Amazon Managed Grafana, Azure Managed Grafana |
| **Logs** | **Loki**, ELK/EFK, OpenSearch, Graylog, Fluent Bit/Fluentd, Vector | Splunk, Datadog Logs, Sumo Logic | CloudWatch Logs, Azure Log Analytics, Cloud Logging |
| **Traces** | **Jaeger**, **Tempo**, Zipkin, SigNoz | Honeycomb, Lightstep, New Relic, Dynatrace | AWS X-Ray, Azure Application Insights, Cloud Trace |
| **Instrumentation standard** | **OpenTelemetry** (CNCF; vendor-neutral APIs, SDKs, Collector) | supported by nearly all vendors | supported |
| **Alert routing / on-call** | **Alertmanager**, Grafana OnCall (check current status) | PagerDuty, Opsgenie (check its current availability), VictorOps/Splunk On-Call | AWS SNS/Chatbot, Azure Action Groups |
| **Profiling** | Pyroscope, Parca | Datadog profiler | CodeGuru Profiler |
| **Synthetic/blackbox** | Blackbox exporter | Pingdom, Checkly, Datadog Synthetics | CloudWatch Synthetics |

(Product availability, names and pricing change often. For example, Grafana's on-call product and Atlassian's Opsgenie have both had announced changes. Say "check current status" in notes and exams rather than asserting permanence.)

## 12.1 Open source vs SaaS (decision table)

| | Self-hosted OSS (Prometheus + Grafana) | SaaS (Datadog etc.) |
|---|---|---|
| Cost model | Infrastructure + engineer time | Per host/GB/metric (can become very expensive) |
| Control/flexibility | Maximum | Limited to vendor features |
| Setup/maintenance | You operate it (scaling, upgrades, HA) | Vendor operates it |
| Vendor lock-in | Low (open standards) | Higher |
| Time to value | Slower | Fast |
| Best for | Cloud-native teams, cost control, learning | Teams wanting speed and less ops |

**Why this unit uses Prometheus + Grafana:** the de-facto open-source standard for cloud-native metrics (Prometheus is a **CNCF graduated** project), free, widely used with Kubernetes, and teaches transferable concepts.

---

# 1️⃣3️⃣ OBSERVABILITY MATURITY LADDER 🪜

| Level | Description |
|---|---|
| **1. Reactive** | Users report outages; manual SSH and `top`; no dashboards |
| **2. Basic monitoring** | Host metrics, uptime checks, some email alerts, noisy |
| **3. Proactive** | Prometheus/Grafana dashboards, alerts with runbooks, centralized logs, golden-signal dashboards |
| **4. Observable** | Metrics + logs + traces correlated (OpenTelemetry), SLOs and error budgets, structured logging, deployment annotations |
| **5. Intelligent/optimizing** | SLO-based alerting (burn rate), automated remediation, anomaly detection, chaos engineering, observability-driven development, cost-aware telemetry |

Move one level at a time. This unit takes you from level 2 to 4.

---

# 1️⃣4️⃣ DRY RUN: AN INCIDENT SOLVED WITH OBSERVABILITY 🕵️

```
14:05  Deployment #482 of checkout-service goes out via the Jenkins pipeline (Unit III)
14:08  ALERT (symptom-based, from Prometheus → Alertmanager → Slack/PagerDuty):
       "CheckoutHighLatency: p95 latency 2.1 s > 0.5 s for 5 min" (SLO burn-rate alert)
14:09  On-call opens the Grafana dashboard (RED): Rate normal, Errors slightly up, Duration p95 jumped at 14:06 ✔ matches the deploy annotation
14:11  Drills from the latency panel into TRACES for slow requests: all slow traces spend 1.8 s in span "inventory-db.query"
14:13  Opens LOGS filtered by trace ID: "slow query: SELECT * FROM stock WHERE sku LIKE '%...' rows scanned=4.2M" (missing index in the new code)
14:15  Decision: roll back deploy #482 (Unit III rollback), or flip the feature flag off
14:18  Latency returns to normal; alert resolves; error budget burned: ~12 minutes
Next day: blameless postmortem → add index, add query-time metric + alert, add a staging load test; update the runbook
MTTD = 3 min (14:05 → 14:08) ; MTTR = 13 min (14:05 → 14:18)
```

Notice: **metrics detected, traces located, logs explained, deployment markers correlated.** This is observability working.

---

# 1️⃣5️⃣ COMMON MISTAKES 🚨

| Mistake | Impact | Fix |
|---|---|---|
| Monitoring only hosts, not user experience | "All green" dashboards while users suffer | Add RED/golden-signal and synthetic checks |
| Alerting on every metric/cause | Alert fatigue | Alert on symptoms/SLOs; dashboards for causes |
| Using averages for latency | Hides tail pain | Percentiles/histograms |
| No `for:` duration / thresholds too tight | Flapping alerts | Use durations and sensible thresholds |
| Dashboards with 60 panels | Nobody understands them | Few, purposeful panels; one question per dashboard |
| Unstructured logs | Impossible to query | JSON structured logs with consistent fields |
| High-cardinality labels (user_id as a metric label) | Memory/cost blow-up | Keep metrics low-cardinality; put IDs in logs/traces |
| No retention/capacity planning | Disk full, lost history | Plan retention, downsample, remote storage |
| Monitoring system not monitored | Silent failure of monitoring | Meta-monitoring, deadman's switch (watchdog alert) |
| No runbooks/owners | Slow response | Runbook links in alert annotations |
| Treating observability as a tool purchase | Buy-in without culture | Instrument well, define SLOs, practice incident reviews |
| Ignoring deployment/change events | Hard to correlate | Annotate dashboards with deploys |
| Storing secrets/PII in logs/metrics | Security/compliance risk | Redact, mask, minimize |

---

# 1️⃣6️⃣ EXAM ANSWER TEMPLATE ✍️

**Q. Differentiate monitoring and observability. Explain the three pillars with examples.**

```
1. Intro: modern distributed systems need both knowing when something breaks and understanding why.
2. Monitoring: predefined metrics/checks/alerts for known failure modes ("is it up?").
3. Observability: ability to infer internal state from telemetry to debug unknown problems ("why?").
4. Table (6-8 rows): question, approach, data, failure types, output, tools, relationship (monitoring ⊂ observability).
5. Three pillars: metrics (numbers over time, Prometheus), logs (events, Loki/ELK), traces (request path, Jaeger/OpenTelemetry) + how they correlate (diagram or incident example).
6. Frameworks: Four Golden Signals/RED/USE; SLI/SLO/SLA.
7. Conclusion: use both; observability reduces MTTD and MTTR.
```

---

# 1️⃣7️⃣ VIVA QUESTIONS (PART 1) 🎯

**Q1. What is monitoring?** Collecting, storing, visualizing and alerting on predefined measurements to know whether systems are healthy.

**Q2. What is observability?** The ability to understand a system's internal state from its external telemetry and debug unknown problems.

**Q3. Monitoring vs observability?** Monitoring answers known questions ("is it working?") with predefined metrics/alerts; observability lets you ask new questions ("why?") using rich correlated telemetry. Monitoring is part of observability.

**Q4. Name the three pillars.** Metrics, logs and traces.

**Q5. Give an example of each pillar.** Metric: `http_requests_total`; log: a JSON error line with a trace ID; trace: a request path across services with span durations.

**Q6. What are the Four Golden Signals?** Latency, traffic, errors, saturation.

**Q7. RED vs USE?** RED (rate, errors, duration) is for services/requests; USE (utilization, saturation, errors) is for resources.

**Q8. Define SLI, SLO, SLA.** SLI is a measured indicator; SLO is the internal target; SLA is the customer contract with consequences.

**Q9. What is an error budget?** 100% minus the SLO: the allowed unreliability used to balance releases and reliability.

**Q10. 99.9% availability allows how much downtime per month?** About 43.8 minutes (8.77 hours per year).

**Q11. Push vs pull monitoring?** Pull: server scrapes targets (Prometheus); push: targets send data to a collector (StatsD/CloudWatch agent); pull makes target health visible via failed scrapes.

**Q12. Black-box vs white-box monitoring?** Black-box probes externally like a user; white-box uses internal instrumentation.

**Q13. What is alert fatigue and how do you avoid it?** Ignoring alerts due to noise; avoid with symptom-based, actionable alerts, `for` durations, grouping and tuning.

**Q14. What are MTTD and MTTR?** Mean time to detect and mean time to recover; observability aims to reduce both.

**Q15. Why use percentiles instead of averages for latency?** Averages hide slow tail requests that affect real users.

**Q16. What is OpenTelemetry?** A CNCF vendor-neutral standard (APIs, SDKs, Collector) for generating and exporting metrics, logs and traces.

---

# 🧠 PART 1 MASTER SHEET

```
MONITORING = predefined metrics/checks/alerts → "Is it broken?" (known unknowns)
OBSERVABILITY = infer internal state from telemetry → "Why?" (unknown unknowns) ; monitoring ⊂ observability
PILLARS: Metrics (numbers/time: Prometheus) | Logs (events: Loki/ELK) | Traces (request path: Jaeger/Tempo/OpenTelemetry)
  + events, profiles, synthetic, RUM ; correlation (trace IDs, exemplars, shared labels) is the superpower
PIPELINE: instrument → collect → process → store → query → visualize → alert → act
WHAT TO MEASURE: Golden Signals (Latency, Traffic, Errors, Saturation) | RED (services) | USE (resources)
SLI → SLO → SLA ; error budget = 100% − SLO ; nines: 99.9% = 43.8 min/month, 99.99% = 4.38 min/month
Black-box (outside, symptoms) vs White-box (inside, causes) ; Pull (Prometheus) vs Push (StatsD, Pushgateway)
Alerts: actionable, urgent, symptom-based, owned ; avoid alert fatigue ; MTTD, MTTA, MTTR, MTBF
Tools: Prometheus + Grafana (+ Alertmanager, Loki, Tempo, OpenTelemetry) vs Datadog/New Relic vs CloudWatch/Azure Monitor
Percentiles not averages ; keep metric labels low-cardinality ; annotate deployments ; monitor the monitors
```

---

## 🫡 PART 1 STATUS

You now know why monitoring exists, how observability differs, the three pillars, what to measure (golden signals, RED, USE), SLIs/SLOs/SLAs, collection models, alerting philosophy and the tool landscape.

**STOP here.** Next up:

> **PART 2 — Prometheus Fundamentals (architecture, data model, metric types, scraping, exporters, instrumenting your own app) 🔥**
