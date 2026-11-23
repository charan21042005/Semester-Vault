# 🔭 UNIT IV — PART 7: ADVANCED OBSERVABILITY STRATEGIES

> **Covered in this document:** from dashboards to a real observability **strategy** (principles) → **logs done right** (structured JSON logging, correlation IDs, a Python JSON-logging example, the log pipeline, **Grafana Loki and LogQL**, Loki vs Elasticsearch/ELK) → **distributed tracing** (traces, spans, context propagation with W3C `traceparent`, sampling, Jaeger/Tempo, service maps) → **OpenTelemetry** (API, SDK, **Collector** pipelines, OTLP, auto vs manual instrumentation, complete Python/Flask examples and a Collector config) → **correlating the pillars** (trace IDs in logs, exemplars, derived fields, shared resource attributes) → **SLO-driven observability** → **synthetic monitoring and RUM** (Core Web Vitals) → **continuous profiling** → **eBPF** and **service-mesh** telemetry → **anomaly detection and AIOps** (z-score PromQL, ML, LLM assistants, risks) → **progressive delivery with metrics** (canary analysis, Argo Rollouts) → **chaos engineering** and game days → incident management and observability-driven development → CI/CD and security observability → telemetry governance (PII, sampling, cost) → an **end-to-end investigation dry run** → an implementation roadmap and checklist → anti-patterns → exam template → viva → master sheet.

---

Absolutely mate. 😤🔥 Parts 2-5 built the **metrics** side (Prometheus + Grafana + alerts). Part 6 scaled it. Part 7 completes your syllabus line: **"implementing advanced observability strategies"**: logs, traces, correlation, SLOs, and the modern practices that make unknown problems debuggable.

---

# 1️⃣ FROM TOOLS TO STRATEGY 🧭

Buying or installing tools is not a strategy. An observability **strategy** answers:

```
1. WHAT matters to users and the business?            → SLIs/SLOs per user journey
2. HOW do we instrument consistently?                  → OpenTelemetry standards, shared libraries, naming/label conventions
3. WHERE does telemetry go?                            → one pipeline (Collector) → metrics, logs, traces backends → Grafana
4. HOW do we connect signals?                          → trace IDs, exemplars, common resource attributes
5. HOW do we act on it?                                → SLO alerts, runbooks, incident process, automation
6. HOW do we keep it sustainable?                      → cost, privacy, ownership, review cycles
```

## 1.1 Seven guiding principles ⭐

```
1. Start from user-facing SLOs, not from tools           5. Standardize on open formats (OpenTelemetry, Prometheus, OTLP)
2. Instrument once, route anywhere (vendor neutrality)   6. Make telemetry cheap to produce, cheap to store, easy to find
3. Correlate: metrics → traces → logs in two clicks      7. Telemetry is a product: owners, quality bar, deprecation, cost budget
4. Alert on symptoms/SLO burn, investigate with detail
```

---

# 2️⃣ LOGS DONE RIGHT 📜⭐⭐

## 2.1 Why logs need structure

```
UNSTRUCTURED:  "Payment failed for user 42 order 9181 after 5s"             ← grep-only, brittle
STRUCTURED:    {"ts":"2026-11-23T10:15:42Z","level":"error","service":"checkout","msg":"payment failed",
                "user_id":42,"order_id":9181,"duration_ms":5000,"trace_id":"7f3a9c2e1b5d4a8f9c0e1d2b3a4f5c6d"}
```

Structured (JSON or logfmt) logs are **machine-parseable**, filterable by field, and can carry **`trace_id`** for correlation.

## 2.2 Logging best practices ✅

| Practice | Detail |
|---|---|
| **Structured JSON to stdout** | 12-factor style: apps write to stdout/stderr; the platform collects (Docker/Kubernetes/journald) |
| **Consistent fields** | `timestamp` (UTC ISO-8601), `level`, `service`, `env`, `version`, `message`, `trace_id`, `span_id`, `request_id` |
| **Levels used properly** | DEBUG (off in prod), INFO (key events), WARN (recoverable), ERROR (needs attention), FATAL |
| **Log events, not noise** | Avoid logging in tight loops; sample noisy logs |
| **Don't log secrets/PII** | Passwords, tokens, card numbers, personal data: redact/mask (Unit V) |
| **Correlation IDs** | A request/trace ID travels through all services and appears in every log line |
| **Cardinality in logs is fine** | Put user IDs/order IDs in logs (not in metric labels) |
| **Retention tiers** | Hot (7-30 d, searchable) → cold/archive (object storage) |

## 2.3 Python JSON logging with trace correlation

```python
import json
import logging
import sys
from datetime import datetime, timezone

from opentelemetry import trace


class JsonFormatter(logging.Formatter):
    def format(self, record):
        ctx = trace.get_current_span().get_span_context()
        entry = {
            "ts": datetime.now(timezone.utc).isoformat(timespec="milliseconds"),
            "level": record.levelname.lower(),
            "service": "flask-app",
            "msg": record.getMessage(),
            "logger": record.name,
        }
        if ctx.is_valid:                                   # attach trace/span IDs when a span is active
            entry["trace_id"] = format(ctx.trace_id, "032x")
            entry["span_id"] = format(ctx.span_id, "016x")
        extra = getattr(record, "fields", None)             # structured extras: logger.info("x", extra={"fields": {...}})
        if extra:
            entry.update(extra)
        if record.exc_info:
            entry["exception"] = self.formatException(record.exc_info)
        return json.dumps(entry)


handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JsonFormatter())
log = logging.getLogger("app")
log.setLevel(logging.INFO)
log.addHandler(handler)

# usage
log.info("order created", extra={"fields": {"order_id": 9181, "user_id": 42, "amount": 499}})
log.error("payment failed", extra={"fields": {"order_id": 9181, "duration_ms": 5000}})
```

(If you haven't installed OpenTelemetry yet, remove the `trace` lines; the logger still works. Libraries like `python-json-logger` or `structlog` also give JSON output.)

## 2.4 The log pipeline 🔄

```
 App (stdout JSON) → node/container runtime → COLLECTOR/AGENT → (parse, enrich, filter, redact, route) → LOG STORE → Grafana/Kibana
                                              Fluent Bit · Fluentd · Vector · Grafana Alloy · OTel Collector · (Promtail: deprecated in favor of Alloy)
                                                                                         Loki · Elasticsearch/OpenSearch · CloudWatch Logs · Splunk
```

## 2.5 Grafana Loki 🪵 ("Prometheus, but for logs")

> **Loki** indexes only **labels** (like Prometheus), not the full text of each log line; log content is stored compressed in object storage. Result: **much cheaper** and simpler than full-text-indexed systems, with PromQL-like querying (**LogQL**).

```
 Collector ──▶ DISTRIBUTOR ──▶ INGESTER (memory + WAL) ──▶ chunks + index in object storage (S3/GCS/filesystem)
 Grafana ──LogQL──▶ QUERY FRONTEND ──▶ QUERIER ──▶ ingesters + storage
```

Labels should be **low-cardinality** (service, namespace, env, level), exactly like Prometheus labels. High-cardinality values (user_id, trace_id) stay **inside the log line** and are filtered at query time.

### LogQL cheat sheet ⭐

```logql
{service="flask-app"}                                   # stream selector (label matchers: = != =~ !~)
{service="flask-app", env="prod"} |= "error"            # line contains "error"
{service="flask-app"} != "health"                       # line does NOT contain "health"
{service="flask-app"} |~ "timeout|refused"              # regex match on the line
{service="flask-app"} | json                            # parse JSON → fields become labels at query time
{service="flask-app"} | json | level="error" | duration_ms > 1000      # filter on parsed fields
{service="flask-app"} | json | line_format "{{.level}} {{.msg}} order={{.order_id}}"   # reformat output
{service="flask-app"} | json | trace_id="7f3a9c2e1b5d4a8f9c0e1d2b3a4f5c6d"            # find all logs of one trace

# METRICS FROM LOGS (LogQL metric queries)
rate({service="flask-app"} |= "error" [5m])                                  # errors per second from logs
sum by (level) (count_over_time({service="flask-app"} | json [1m]))          # log volume by level
topk(5, sum by (order_id) (count_over_time({service="flask-app"} | json | level="error" [1h])))
quantile_over_time(0.95, {service="flask-app"} | json | unwrap duration_ms [5m]) by (service)
```

Quick lab start (optional): `docker run -d --name loki -p 3100:3100 grafana/loki` then add a **Loki** data source in Grafana (`http://<host>:3100`) and ship logs with Grafana Alloy or another agent (check the current Loki docs for the recommended agent configuration, since collectors and their config syntax have changed recently).

## 2.6 Loki vs Elasticsearch (ELK/EFK) ⭐

| | **Loki** | **Elasticsearch/OpenSearch (ELK/EFK)** |
|---|---|---|
| Indexing | Labels only; content grep at query time | Full-text inverted index of all fields |
| Storage cost | **Low** (compressed chunks in object storage) | Higher (index overhead) |
| Query power | Good for known streams; scanning for text | Excellent full-text search, aggregations, analytics |
| Operations | Simpler, cloud-native, Grafana-integrated | Heavier cluster to tune (shards, heap) |
| Best for | Operational logs correlated with metrics/traces | Search-heavy logging, SIEM, analytics |
| UI | Grafana | Kibana/OpenSearch Dashboards |

---

# 3️⃣ DISTRIBUTED TRACING 🧵⭐⭐

## 3.1 Concepts

| Term | Meaning |
|---|---|
| **Trace** | The full journey of **one request** through all services (identified by a **trace ID**) |
| **Span** | One unit of work inside a trace (an HTTP call, DB query, function) with **start time, duration, attributes, status**; has a **span ID** and a **parent span ID** |
| **Root span** | The first span (entry point) |
| **Attributes (tags)** | Key-value metadata: `http.method`, `db.statement`, `user.tier` |
| **Events** | Timestamped annotations inside a span (exception, retry) |
| **Links** | Relationship to other spans/traces (async/batch work) |
| **Span kind** | SERVER, CLIENT, PRODUCER, CONSUMER, INTERNAL |
| **Context propagation** | Passing trace/span IDs between services (HTTP headers, message headers) so spans join into one trace |
| **Baggage** | Key-values propagated with the request (use sparingly) |
| **Sampling** | Deciding which traces to keep (cost control) |

## 3.2 Context propagation: W3C Trace Context ⭐

```
traceparent: 00-7f3a9c2e1b5d4a8f9c0e1d2b3a4f5c6d-00f067aa0ba902b7-01
             │  │                                │                │
             │  trace-id (16 bytes)               parent span-id   flags (01 = sampled)
             version
```

Each service reads `traceparent` from the incoming request, creates a **child span**, and forwards an updated header on outgoing calls. OpenTelemetry instrumentation libraries do this automatically for common frameworks (Flask, requests, Django, Spring, Express...).

## 3.3 Sampling strategies ⭐

| Strategy | How | Pros/Cons |
|---|---|---|
| **Always on** | Keep every trace | Complete but expensive |
| **Head-based (probabilistic)** | Decide at the start: keep 5-10% | Cheap, simple; may drop interesting (error/slow) traces |
| **Tail-based** | Collector buffers the whole trace, then keeps **errors, slow traces and a baseline sample** | Best value; needs Collector memory and routing by trace ID |
| **Adaptive/rate-limited** | Keep N traces per second per service | Protects backends under spikes |

## 3.4 Backends and tools

| Tool | Notes |
|---|---|
| **Jaeger** | CNCF graduated; created at **Uber** (Unit I case study); UI for trace search and dependency graphs |
| **Grafana Tempo** | Cost-efficient trace storage in object storage; **TraceQL**; integrates with Loki/Prometheus (exemplars, trace-to-logs/metrics) |
| **Zipkin** | Original open-source tracer |
| **AWS X-Ray, Azure Application Insights, Google Cloud Trace** | Cloud-native tracing |
| **Honeycomb, Lightstep, Datadog APM, New Relic, Dynatrace** | Commercial platforms |

## 3.5 What traces give you

```
✔ Latency breakdown per service/dependency (flame graph / waterfall)    ✔ Service dependency map (auto-generated)
✔ Error origin (which span failed)                                       ✔ Metrics from traces (span metrics → RED per endpoint/dependency)
✔ Compare slow vs normal traces (what differs?)                          ✔ Critical-path analysis
```

---

# 4️⃣ OPENTELEMETRY (OTel) ⭐⭐⭐

> **OpenTelemetry** is a **CNCF** project (formed in 2019 by merging OpenTracing and OpenCensus) providing **vendor-neutral APIs, SDKs, tools and a Collector** to generate, process and export **traces, metrics and logs** (profiles are emerging). It is now the de-facto standard for instrumentation.

## 4.1 Components

| Component | Role |
|---|---|
| **API** | Stable interfaces used by your code/libraries (`tracer.start_span`) |
| **SDK** | Implementation: sampling, batching, resource info, exporters |
| **Instrumentation libraries / auto-instrumentation** | Ready-made hooks for frameworks (Flask, Django, requests, SQLAlchemy, Spring...) |
| **OTLP** | **OpenTelemetry Protocol**: standard wire format; gRPC **4317**, HTTP **4318** |
| **Collector** | A standalone service/agent: **receives**, **processes**, **exports** telemetry (the "telemetry router") |
| **Semantic conventions** | Standard names for attributes (`service.name`, `http.request.method`, `db.system`) so data is consistent |
| **Resource attributes** | Describe the source: `service.name`, `service.version`, `deployment.environment`, `k8s.pod.name`, `cloud.region` |

## 4.2 Architecture

```
 App (OTel SDK + auto-instrumentation) ──OTLP──▶ OTEL COLLECTOR ─────────┬──▶ Traces  → Tempo / Jaeger
 App2, App3, Kubernetes, hosts, DBs (receivers) ──────────────▶ pipelines ├──▶ Metrics → Prometheus (scrape :8889 or remote_write) / Mimir
                                                                (receivers → processors → exporters) └──▶ Logs → Loki / Elasticsearch
 Deployment patterns:  AGENT (per host/pod sidecar/DaemonSet) → GATEWAY (central, tail sampling, routing) → backends
```

Why use the Collector? **Decouple** apps from backends: change vendors without redeploying apps; add **sampling, batching, redaction, enrichment, retries, multi-destination routing** centrally.

## 4.3 Collector configuration ⭐ (`otel-collector.yaml`)

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:                 # protect the Collector from OOM
    check_interval: 1s
    limit_mib: 512
  batch: {}                       # batch telemetry for efficient export
  attributes/redact:              # example: remove a sensitive attribute
    actions:
      - key: user.email
        action: delete

exporters:
  otlp/tempo:                     # traces → Tempo
    endpoint: tempo:4317
    tls:
      insecure: true
  prometheus:                     # metrics → exposed for Prometheus to scrape at :8889/metrics
    endpoint: 0.0.0.0:8889
  debug:                          # print telemetry to the Collector log (great for learning/debugging)
    verbosity: basic

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, attributes/redact, batch]
      exporters: [otlp/tempo, debug]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

Run (container):

```bash
docker run -d --name otelcol -p 4317:4317 -p 4318:4318 -p 8889:8889 \
  -v $(pwd)/otel-collector.yaml:/etc/otelcol-contrib/config.yaml \
  otel/opentelemetry-collector-contrib
```

(Image names, default config paths and available components depend on the distribution/version: check the OpenTelemetry docs. The "contrib" distribution includes the most exporters/processors.)

## 4.4 Instrumenting a Flask app: manual + automatic 🐍

Install:

```bash
pip install flask opentelemetry-sdk opentelemetry-exporter-otlp \
            opentelemetry-instrumentation-flask opentelemetry-instrumentation-requests
```

`app_otel.py`:

```python
import random
import time

from flask import Flask, jsonify
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# 1) describe WHO is producing telemetry (resource attributes)
resource = Resource.create({
    "service.name": "flask-app",
    "service.version": "1.0.0",
    "deployment.environment": "dev",
})

# 2) configure the tracer provider + OTLP exporter to the Collector
provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("flask-app")

app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)      # AUTOMATIC spans for every request (+ propagates traceparent)
RequestsInstrumentor().instrument()          # AUTOMATIC spans for outgoing `requests` calls


@app.route("/checkout")
def checkout():
    with tracer.start_as_current_span("validate-cart") as span:          # MANUAL child span
        span.set_attribute("cart.items", 3)
        time.sleep(random.uniform(0.01, 0.05))

    with tracer.start_as_current_span("charge-card") as span:
        span.set_attribute("payment.provider", "demo-bank")
        delay = random.uniform(0.05, 0.9)
        time.sleep(delay)
        if delay > 0.8:                                                   # record an error on the span
            span.set_status(trace.StatusCode.ERROR, "gateway timeout")
            span.add_event("retry", {"attempt": 1})
            return jsonify(error="payment timeout"), 502
    return jsonify(status="paid")


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### Zero-code auto-instrumentation (alternative)

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install            # installs instrumentations for the libraries it finds
export OTEL_SERVICE_NAME=flask-app
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_TRACES_EXPORTER=otlp OTEL_METRICS_EXPORTER=otlp OTEL_LOGS_EXPORTER=otlp
opentelemetry-instrument python app.py       # no code changes
```

Standard environment variables (`OTEL_*`) configure SDKs in any language. Java uses a `-javaagent:opentelemetry-javaagent.jar` flag; Node.js, .NET, Go have similar packages.

## 4.5 OTel vs Prometheus client libraries (when each?)

| | Prometheus client libs | OpenTelemetry SDK |
|---|---|---|
| Signals | Metrics only | Traces + metrics + logs |
| Backend | Prometheus-format `/metrics` (pull) | OTLP (push) → Collector → any backend |
| Ecosystem | Mature, simple | Fast-growing standard; Prometheus accepts OTLP and the Collector can expose Prometheus format |
| Choice | Metrics-only apps, simplicity | Polyglot/microservices wanting one standard for all signals |

They coexist: many teams keep Prometheus metrics and add OTel for traces/logs.

---

# 5️⃣ CORRELATING THE PILLARS ⭐⭐⭐ (THE SUPERPOWER)

Without correlation you have three separate silos; with it you move **metric → trace → log** in seconds.

| Technique | How it works |
|---|---|
| **Trace ID in every log line** | Logging formatter adds `trace_id` (Section 2.3) → Loki query `| json | trace_id="..."` |
| **Exemplars** | A metric sample carries a **trace ID** of an example request (e.g., a slow request in a latency histogram bucket); Grafana shows dots on the graph that link to the trace. Needs OpenMetrics exposition + Prometheus exemplar storage (a feature flag in many versions: check your version) + client library support |
| **Grafana derived fields / trace-to-logs / trace-to-metrics** | Configure the Loki data source to turn `trace_id=` in log lines into a link to Tempo; Tempo data source links back to logs/metrics by service and time |
| **Shared resource attributes/labels** | The same `service`/`env`/`version` values in metrics labels, log labels and span attributes, so you can pivot with a click |
| **Span metrics → RED metrics** | Collector/Tempo generates request rate/error/duration metrics from spans (no extra instrumentation) |
| **Deployment/event annotations** | Release markers on every dashboard |

## 5.1 Example drill-down workflow

```
1. SLO burn alert fires (Prometheus): "checkout p95 latency high"
2. Open Grafana RED dashboard → latency panel shows spike; exemplar dots on the histogram
3. Click an exemplar → Tempo trace opens → the "charge-card" span takes 1.8 s with attribute payment.provider=demo-bank
4. Click "Logs for this span" → Loki shows: {"level":"error","msg":"gateway 503","trace_id":"...","retry":3}
5. Conclusion: dependency problem, not our code → mitigate (feature flag fallback/circuit breaker) → postmortem
```

---

# 6️⃣ SLO-DRIVEN OBSERVABILITY ⭐⭐

```
1. Identify CRITICAL USER JOURNEYS (browse, search, checkout, login)
2. Define SLIs per journey (availability = good requests / total; latency = % under threshold)
3. Set SLOs with stakeholders (e.g., checkout availability 99.9%, p95 < 500 ms) and an ERROR BUDGET POLICY
4. Implement: recording rules (Part 3) + multi-window burn-rate alerts (Part 4) + SLO dashboards (Part 5)
5. Review monthly: attainment, budget burn, top contributors; decide (ship features vs invest in reliability)
6. Tie to delivery: freeze risky releases when the budget is exhausted (Unit III), celebrate healthy budgets
```

Generate rules/dashboards from a compact SLO spec with **Sloth** or **Pyrra**:

```yaml
# Sloth-style SLO spec (conceptual)
version: "prometheus/v1"
service: "flask-app"
slos:
  - name: "requests-availability"
    objective: 99.9
    sli:
      events:
        error_query: sum(rate(myapp_http_requests_total{status=~"5.."}[{{.window}}]))
        total_query: sum(rate(myapp_http_requests_total[{{.window}}]))
    alerting:
      name: FlaskAvailability
      page_alert:   { labels: { severity: critical } }
      ticket_alert: { labels: { severity: warning } }
```

SLO dashboard panels: current SLI, budget remaining (%), burn rate, time-to-exhaustion, top error sources.

---

# 7️⃣ SYNTHETIC MONITORING AND REAL USER MONITORING (RUM) 🌐

| | **Synthetic monitoring** | **RUM (Real User Monitoring)** |
|---|---|---|
| What | **Scripted probes** simulating users from chosen locations on a schedule | **Telemetry from real users' browsers/apps** |
| Strength | Detects outages even at 3 AM with no traffic; consistent baseline; tests from multiple regions | Shows true experience across devices/networks/geographies |
| Weakness | Not real users; limited coverage | Needs traffic; privacy considerations |
| Tools | Blackbox exporter, Grafana Synthetic Monitoring, Checkly, Pingdom, CloudWatch Synthetics, k6/Playwright scripts | Grafana Faro, OpenTelemetry browser, Datadog RUM, Google Analytics/CrUX |
| Metrics | `probe_success`, `probe_duration_seconds`, SSL expiry, step timings | **Core Web Vitals**: **LCP** (loading), **INP** (interactivity; replaced FID), **CLS** (visual stability), TTFB, JS errors |

Synthetic transaction example (k6 browser/Playwright idea): "open site → search → add to cart → checkout (test account)" every 5 minutes from Mumbai, Frankfurt and Virginia; alert on `probe_success == 0` from ≥ 2 locations.

Rule: **combine both**: synthetic = availability guard; RUM = experience truth.

---

# 8️⃣ CONTINUOUS PROFILING 🔥

> **Profiling** shows **which functions consume CPU/memory** (flame graphs). **Continuous profiling** does it always-on in production with low overhead, linked to traces/metrics.

| Tool | Notes |
|---|---|
| **Grafana Pyroscope** | Open-source continuous profiling; Grafana integration |
| **Parca** | eBPF-based open-source profiler |
| **Cloud profilers** | AWS CodeGuru Profiler, Google Cloud Profiler |

Use when metrics say "CPU high" and traces say "this function is slow" but you need to know **which lines/functions** burn time. Flame graph: wide bar = more time in that function.

---

# 9️⃣ eBPF AND SERVICE-MESH OBSERVABILITY 🐝

## 9.1 eBPF

> **eBPF** lets small, sandboxed programs run **inside the Linux kernel** to observe system calls, network packets and function calls **without changing application code** and with low overhead.

| Tool | Use |
|---|---|
| **Pixie** | Instant Kubernetes observability (requests, SQL, flame graphs) |
| **Cilium Hubble** | Network flows and service dependencies |
| **Grafana Beyla / OpenTelemetry eBPF projects** | Auto-instrument HTTP/gRPC to produce RED metrics and traces |
| **Parca/Pyroscope eBPF profilers** | System-wide profiling |
| **Falco** | Runtime **security** detection (Unit V) |

Pros: zero-code, language-agnostic, covers legacy apps. Cons: needs modern Linux kernels/privileges, limited business context (no custom attributes), harder to see inside encrypted payloads/app logic. **Best as a complement to SDK instrumentation.**

## 9.2 Service mesh / proxies

Istio, Linkerd, Envoy sidecars emit **golden-signal metrics, access logs and traces** for every service-to-service call with **no code changes** (success rate, latency percentiles per route), plus mTLS and traffic policies. Trade-off: extra complexity and resource overhead.

---

# 🔟 ANOMALY DETECTION AND AIOps 🤖

## 10.1 Why

Static thresholds fail for metrics with **seasonality** (daily/weekly patterns) or when you don't know "normal". Anomaly detection learns the baseline.

## 10.2 Simple, effective: z-score in PromQL ⭐

```promql
# flag when the request rate is > 3 standard deviations from its own 1-day average
abs(
  sum(rate(myapp_http_requests_total[5m]))
  - avg_over_time(sum(rate(myapp_http_requests_total[5m]))[1d:5m])
)
>
3 * stddev_over_time(sum(rate(myapp_http_requests_total[5m]))[1d:5m])
```

Better for weekly patterns: compare with the same time last week (`offset 7d`) and use recording rules for the baseline and spread.

```promql
sum(rate(myapp_http_requests_total[5m])) / sum(rate(myapp_http_requests_total[5m] offset 1w)) < 0.5    # traffic halved vs last week
```

## 10.3 Techniques and tools

| Approach | Examples |
|---|---|
| **Statistical** | Z-score, moving averages, seasonal decomposition, EWMA, Holt-Winters |
| **Machine learning** | Isolation Forest, Prophet/ARIMA forecasting, autoencoders, vendor ML (Datadog Watchdog, Dynatrace Davis, CloudWatch Anomaly Detection alarms, Grafana ML features) |
| **Alert correlation/noise reduction** | Group related alerts into one incident, topology-aware correlation |
| **Root-cause suggestions** | Rank changes (deploys, config, flags) co-occurring with the anomaly |
| **LLM/AI assistants** | Draft PromQL/LogQL from natural language, summarize incidents and logs, suggest runbook steps (always verify) |

## 10.4 Risks and guardrails ⚠️

```
• False positives/negatives: models need tuning; seasonal events (festival sales) look "anomalous"
• Explainability: engineers must trust and understand why an alert fired → keep human-readable reasons
• Data quality: garbage in, garbage out; gaps and label churn break baselines
• Over-automation: don't auto-remediate dangerous actions without safeguards, approvals and rollbacks
• Privacy: don't send sensitive telemetry to third-party AI services without governance
• Keep SLO/symptom alerts as the backbone; use AIOps to reduce noise and speed triage, not to replace fundamentals
```

---

# 1️⃣1️⃣ PROGRESSIVE DELIVERY DRIVEN BY METRICS 🚦 (LINKS TO UNIT III)

Observability closes the loop on **canary** and **blue-green** deployments: promotion or rollback is **decided by metrics**, automatically.

```
Deploy 5% canary → analysis queries Prometheus (error rate, latency vs baseline) for N minutes
   healthy → 25% → 50% → 100%        unhealthy → automatic rollback + alert
```

Tools: **Argo Rollouts**, **Flagger**, Spinnaker/Kayenta (Netflix-origin automated canary analysis), AWS CodeDeploy with CloudWatch alarms.

Argo Rollouts `AnalysisTemplate` (Prometheus provider):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
    - name: error-rate
      interval: 1m
      count: 5
      failureLimit: 1
      successCondition: result[0] < 0.05          # fail the rollout if > 5% errors
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(myapp_http_requests_total{status=~"5..", version="canary"}[2m]))
            / sum(rate(myapp_http_requests_total{version="canary"}[2m]))
```

Prerequisite: metrics labeled with `version`/`revision` so canary vs stable can be compared (add the label via relabeling or app metadata).

---

# 1️⃣2️⃣ CHAOS ENGINEERING AND GAME DAYS 🐒⭐

> **Chaos engineering** = deliberately injecting failures in a controlled way to **verify that the system (and your observability and runbooks) handle them**. Popularized by **Netflix's Chaos Monkey** (Unit I Part 7).

## 12.1 Process

```
1. Define STEADY STATE with SLIs ("checkout success ≥ 99.9%, p95 < 500 ms")
2. Hypothesis: "If one web instance dies, users see no impact and an alert fires within 2 minutes"
3. Inject failure with a SMALL blast radius (one instance/pod, a staging env first)
4. OBSERVE: did the SLIs hold? did alerts fire? did dashboards/traces/logs make diagnosis easy? did auto-scaling/failover work?
5. Learn and fix gaps (missing metric, wrong threshold, absent runbook); automate and repeat; widen scope gradually
```

## 12.2 Failure types and tools

| Failure | Example tool |
|---|---|
| Kill instances/pods | Chaos Monkey, **Chaos Mesh**, **LitmusChaos**, `kubectl delete pod` |
| Network latency/packet loss | `tc netem`, Chaos Mesh, Toxiproxy, Gremlin |
| CPU/memory/disk stress | `stress-ng`, Gremlin, AWS **Fault Injection Service (FIS)** |
| Dependency outage/region failure | FIS, Gremlin, simulated DNS failures |

**Game day:** a scheduled, cross-team exercise where failures are injected and the team practices detection, communication and recovery. **Observability is what makes chaos useful**: without it you can't tell what happened.

⚠️ Always: get approval, define abort conditions ("stop if error budget burn > X"), start in non-prod, avoid peak business hours, notify on-call.

---

# 1️⃣3️⃣ INCIDENT MANAGEMENT AND OBSERVABILITY-DRIVEN DEVELOPMENT 🚨🧑‍💻

## 13.1 Incident lifecycle with observability

```
DETECT (SLO/symptom alert) → TRIAGE (severity, dashboards) → DIAGNOSE (traces/logs/profiles, recent changes via annotations)
→ MITIGATE (rollback, flag off, scale, failover) → COMMUNICATE (status page, Slack incident channel) → RESOLVE → POSTMORTEM (blameless) → IMPROVE
```

Good practice: alert → **dashboard link** + **runbook link**; incident commander role; ChatOps commands (`/incident`, deploy rollback); status page; timeline built from telemetry; postmortem action items tracked (new alerts, tests, runbooks).

## 13.2 Observability-driven development (ODD)

```
• Instrument WHILE coding: every feature ships with metrics, structured logs and spans, plus a dashboard panel and an alert/SLO if user-facing
• Definition of Done (Unit III) includes observability ("can we tell if this works in production?")
• Use feature flags + metrics for safe experiments (A/B guardrails from Unit III Part 6)
• Write the question first: "How will we know this is broken?" then add the signal
• Test observability: unit-test metrics/logs; promtool rule tests; chaos/game days verify alerts
```

---

# 1️⃣4️⃣ CI/CD AND SECURITY OBSERVABILITY 🔗

| Area | What to observe |
|---|---|
| **Pipelines (Unit III)** | Jenkins **Prometheus metrics plugin** (`/prometheus`): build duration, queue time, failure rate, executor usage; Jenkins **OpenTelemetry plugin** (pipeline traces); DORA metrics dashboards; deployment annotations on Grafana |
| **Infrastructure/IaC (Unit II)** | Drift detection results, apply failures, resource counts, cost metrics |
| **Security (Unit V)** | Audit logs (CloudTrail, Kubernetes audit), auth failures, anomaly detection, **Falco** runtime alerts, vulnerability trends, secrets-access logs; centralize in a SIEM |
| **Cost (Unit VI)** | Cost per service/team, telemetry spend, budget alerts; unit economics (cost per 1,000 requests) |

Example Jenkins Prometheus scrape job:

```yaml
- job_name: "jenkins"
  metrics_path: /prometheus
  static_configs:
    - targets: ["jenkins.example.com:8080"]
```

---

# 1️⃣5️⃣ TELEMETRY GOVERNANCE: PRIVACY, SAMPLING, RETENTION 🔐💰

```
PRIVACY:   redact PII/secrets at the source or in the Collector (attributes/redact, transform processors); avoid user emails/IDs in metric labels;
           comply with GDPR/DPDP-style rules: purpose limitation, retention limits, access control, right to erasure for logs where required
SAMPLING:  head-based for volume, tail-based for value (keep errors/slow), always keep 100% of metrics; document sampling rates so analysts aren't misled
RETENTION: hot (days) → warm (weeks) → cold/object storage (months-years) ; downsample metrics ; archive logs cheaply
ACCESS:    RBAC per team/tenant ; audit who queries sensitive logs ; separate prod/non-prod
COST:      per-team telemetry budgets ; weekly "unused metrics/dashboards/alerts" review ; cardinality limits ; drop DEBUG in prod
OWNERSHIP: every dashboard/alert/metric family has an owner and a purpose; deprecate regularly
```

---

# 1️⃣6️⃣ END-TO-END INVESTIGATION (DRY RUN) 🕵️

Scenario: *"Checkout is slow for some users since 14:05."* (extends Part 1)

```
14:08  Alertmanager → Slack: "ErrorBudgetBurnFast: checkout p95 > 1.5 s; 14.4× burn" (SLO alert, Part 4)
14:09  Grafana RED dashboard (Part 5): p95 jumped at 14:06; deploy annotation "build #482 (flask-app v1.8.0)" at 14:05
14:10  Metric breakdown by version label: only v1.8.0 pods slow; v1.7.9 normal → release-related
14:11  Exemplar on the latency histogram → Tempo trace: span "charge-card" 1.8 s; child span "db.query stock" 1.7 s (SELECT with LIKE, 4.2M rows)
14:12  Click "logs for span" → Loki: {"level":"warn","msg":"slow query","rows_scanned":4200000,"trace_id":"…","query":"…LIKE '%x%'"}
14:13  Profile (Pyroscope) confirms CPU in the new search function; code review shows missing index in migration
14:14  Mitigation: Argo Rollouts aborts the canary / flag `new_search` turned OFF → traffic back on v1.7.9
14:18  p95 normal; alert resolves; SLO budget used ≈ 12 min
Next day: add index, add slow-query metric + alert, add load test in the Jenkins pipeline (Unit III), update runbook, blameless postmortem
```

Metrics detected, traces located, logs explained, profiles confirmed, deployment data correlated, rollout automation mitigated.

---

# 1️⃣7️⃣ IMPLEMENTATION ROADMAP AND CHECKLIST 🗺️✅

| Phase | Actions | Outcome |
|---|---|---|
| **0. Foundations** | Naming/label conventions, service catalog (owners), `env/service/version` labels | Consistent telemetry |
| **1. Metrics + alerts** | Prometheus + Node Exporter + app RED metrics; Grafana dashboards; Alertmanager routing (Parts 2-5) | Basic monitoring |
| **2. Logs** | JSON logging, central pipeline, Loki/ELK, retention | Searchable events |
| **3. Traces** | OpenTelemetry SDK/auto-instrumentation, Collector, Tempo/Jaeger, sampling | Request-level visibility |
| **4. Correlation** | trace_id in logs, exemplars, Grafana links, shared attributes, deploy annotations | Two-click diagnosis |
| **5. SLOs** | SLIs/SLOs, burn-rate alerts, SLO dashboards, error budget policy | User-centric alerting |
| **6. Scale + HA** | HA pairs, Thanos/Mimir or managed, cardinality controls, meta-monitoring (Part 6) | Reliable, scalable platform |
| **7. Proactive** | Synthetic/RUM, canary analysis, anomaly detection, chaos/game days | Find problems before users |
| **8. Culture + governance** | ODD, blameless postmortems, cost/privacy governance, training, on-call health | Sustainable practice |

Checklist:

```
☐ Every service exposes RED metrics, structured logs with trace_id, and traces via OTel
☐ One Collector pipeline per environment; backends: Prometheus/Mimir + Loki + Tempo + Grafana (or equivalents)
☐ SLOs for critical journeys with burn-rate alerts and runbooks; alerts routed by team/severity
☐ Deployment annotations on dashboards; canary analysis tied to metrics
☐ Synthetic probes for key journeys; RUM Core Web Vitals
☐ HA + remote storage + meta-monitoring (Watchdog); backups of dashboards/rules (Git)
☐ PII redaction, sampling policy, retention tiers, cost budget per team
☐ Regular game days; blameless postmortems feeding improvements
```

---

# 1️⃣8️⃣ ANTI-PATTERNS 🚨

| Anti-pattern | Why bad | Better |
|---|---|---|
| "Three pillars" as three disconnected tools | No correlation, slow diagnosis | trace IDs, exemplars, shared labels |
| Logging everything at DEBUG in prod | Cost and noise | Sampling, levels, structured essentials |
| Metrics with user_id/request_id labels | Cardinality explosion | IDs in logs/traces |
| 100% trace sampling forever | Cost | Tail/head sampling strategy |
| Vendor-specific instrumentation everywhere | Lock-in | OpenTelemetry + Collector |
| Dashboards without SLOs/owners | Eyeball monitoring | SLO-driven alerting, owned dashboards |
| Alerting on causes only | Noise | Symptom/SLO burn alerts |
| AI auto-remediation without guardrails | Risky automation | Human approval, rollback, audit |
| No observability in Definition of Done | Blind releases | ODD, instrumentation reviews |
| Ignoring privacy | Compliance/security risk | Redaction, retention, access control |

---

# 1️⃣9️⃣ EXAM ANSWER TEMPLATE ✍️

**Q. Explain advanced observability strategies for a microservices application.**

```
1. Intro: observability = understanding unknown problems from telemetry; strategy = SLOs + instrumentation + correlation + action.
2. Three pillars refresher with tools (Prometheus, Loki/ELK, Tempo/Jaeger) + diagram of the pipeline (apps → OTel Collector → backends → Grafana).
3. OpenTelemetry: API/SDK/Collector, OTLP, auto vs manual instrumentation, benefits (vendor-neutral).
4. Structured logging with trace IDs; distributed tracing with context propagation and sampling.
5. Correlation: exemplars, derived fields, shared attributes, deployment annotations.
6. SLO-based alerting with burn rates; synthetic monitoring + RUM.
7. Advanced: continuous profiling, eBPF, service mesh telemetry, anomaly detection/AIOps (with limits).
8. Progressive delivery with metric-based canary analysis; chaos engineering/game days.
9. Governance: privacy, sampling, retention, cost; meta-monitoring.
10. Example incident drill-down; conclusion (reduces MTTD/MTTR, improves reliability).
```

---

# 2️⃣0️⃣ VIVA QUESTIONS (PART 7) 🎯

**Q1. What is structured logging and why use it?** Emitting logs as JSON/logfmt with consistent fields so they can be parsed, filtered and correlated (e.g., with `trace_id`).

**Q2. How does Loki differ from Elasticsearch?** Loki indexes only labels and stores compressed log chunks cheaply; Elasticsearch builds a full-text index, offering richer search at higher cost.

**Q3. Give three LogQL examples.** `{service="app"} |= "error"`, `{service="app"} | json | level="error"`, `rate({service="app"} |= "error" [5m])`.

**Q4. What are a trace and a span?** A trace is one request's journey across services; a span is one timed operation within it with parent-child relationships.

**Q5. What is context propagation?** Passing trace/span IDs between services (W3C `traceparent` header) so spans join into one trace.

**Q6. What is OpenTelemetry?** A CNCF vendor-neutral standard (APIs, SDKs, Collector, OTLP) for generating and exporting traces, metrics and logs.

**Q7. What does the OTel Collector do?** Receives telemetry, processes it (batch, sample, redact, enrich) and exports to one or more backends, decoupling apps from vendors.

**Q8. Head-based vs tail-based sampling?** Head decides at the start (cheap, may miss rare errors); tail decides after the trace completes (keeps errors/slow traces, needs more resources).

**Q9. What are exemplars?** Trace IDs attached to metric samples so you can jump from a metric spike to an example trace.

**Q10. How do you correlate metrics, logs and traces?** Use common labels/attributes, trace IDs in logs, exemplars, Grafana derived fields and trace-to-logs links.

**Q11. What is SLO-based alerting?** Alerting on error-budget burn rate (multi-window) tied to user-facing SLIs instead of low-level thresholds.

**Q12. Synthetic monitoring vs RUM?** Synthetic uses scripted probes to test availability proactively; RUM measures real users' experience (Core Web Vitals).

**Q13. What is eBPF-based observability?** Kernel-level programs that capture network/system/app activity without code changes and with low overhead; complements SDK instrumentation.

**Q14. What is chaos engineering?** Controlled failure injection to verify resilience and the effectiveness of monitoring and runbooks, e.g., Chaos Monkey.

**Q15. How can AI/ML help observability and what are the risks?** Anomaly detection, alert correlation, root-cause hints and query/summary assistance; risks include false positives, poor explainability, bad data and unsafe automation.

**Q16. What is observability-driven development?** Instrumenting features as they are built, with dashboards, alerts/SLOs and observability in the Definition of Done.

---

# 🧠 PART 7 MASTER SHEET

```
STRATEGY: SLO-first • instrument once (OpenTelemetry) • route anywhere (Collector) • correlate • alert on symptoms/burn • govern cost/privacy
LOGS: structured JSON to stdout ; fields (ts, level, service, env, trace_id) ; no PII ; pipeline: app → agent (Fluent Bit/Vector/Alloy/OTel) → Loki/ELK → Grafana
LOKI: label-only index, chunks in object storage ; LogQL: {labels} |= "x" | json | field filter ; rate()/count_over_time() ; vs Elasticsearch (full-text, costlier)
TRACING: trace → spans (parent/child) ; attributes/events/links ; W3C traceparent (version-traceid-spanid-flags) ; sampling head/tail ; Jaeger (Uber), Tempo (TraceQL)
OPENTELEMETRY: API + SDK + auto-instrumentation + Collector (receivers → processors → exporters) ; OTLP 4317 gRPC / 4318 HTTP ; semantic conventions ; resource attributes
CORRELATION: trace_id in logs • exemplars • Grafana derived fields/trace-to-logs • shared service/env/version labels • span metrics • deploy annotations
SLO: SLIs → SLOs → error budget policy → burn-rate alerts (14.4×/6×/1×) → SLO dashboards ; Sloth/Pyrra
SYNTHETIC (probes, blackbox/k6/Checkly) + RUM (Core Web Vitals: LCP, INP, CLS) ; PROFILING (Pyroscope/Parca, flame graphs) ; eBPF (Pixie, Hubble, Beyla, Falco) ; service mesh telemetry
ANOMALY/AIOps: z-score PromQL, seasonality (offset 1w), ML, alert correlation, LLM helpers ; risks: false positives, explainability, unsafe automation
PROGRESSIVE DELIVERY: canary/blue-green gated by Prometheus metrics (Argo Rollouts, Flagger, Kayenta)
CHAOS: steady state → hypothesis → small blast radius → observe → learn ; Chaos Monkey/Mesh/Litmus/FIS ; game days
INCIDENTS + ODD: detect → triage → diagnose → mitigate → communicate → postmortem ; observability in Definition of Done
GOVERNANCE: redact PII, sampling policy, retention tiers, RBAC, cost budgets, ownership
```

---

## 🫡 PART 7 STATUS

You can design an observability strategy: structured logs with Loki/LogQL, distributed tracing with OpenTelemetry, correlated signals, SLO-driven alerting, synthetic/RUM, profiling, eBPF, anomaly detection, metric-gated delivery, chaos engineering and telemetry governance.

**STOP here.** Next up:

> **PART 8 — Practical Lab Guide (Prometheus + Node Exporter + alert rules, then Grafana dashboards), step by step 🧪**
