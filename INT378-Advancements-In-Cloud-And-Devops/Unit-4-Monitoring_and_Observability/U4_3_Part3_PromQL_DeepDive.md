# 🧮 UNIT IV — PART 3: PromQL DEEP DIVE (QUERYING PROMETHEUS)

> **Covered in this document:** what PromQL is and the four **data types** (instant vector, range vector, scalar, string) → **selectors** and label matchers → range selectors, `offset` and `@` → how evaluation works (instant vs range queries, step, 5-minute staleness) → **operators** (arithmetic, comparison with and without `bool`, logical `and/or/unless`, vector matching with `on`, `ignoring`, `group_left`) → **aggregation** (`sum`, `avg`, `count`, `topk`, `quantile`, `by`/`without`) → **functions** (`rate` vs `irate` vs `increase`, `delta`, `predict_linear`, `*_over_time`, `absent`, `label_replace`, `clamp`, `round`, time functions) → **histograms** (`histogram_quantile`, average latency, fraction under SLO, Apdex) → **subqueries** → **the full Node Exporter query cookbook** (CPU, load, memory, swap, disk space, disk I/O, network, uptime, `up`) for Practicals 8 and 9 → **RED queries for your Flask app** → **recording rules** (why, naming, files, `promtool`) → SLO/availability queries → Prometheus self-monitoring queries → common mistakes and debugging method → 12 practice questions with answers → viva → master sheet.

---

Absolutely mate. 😤🔥 Part 2 collected the data. Part 3 teaches you to **ask questions of it**: **PromQL (Prometheus Query Language)**. Alerts (Part 4) and Grafana dashboards (Part 5) are both just PromQL expressions, so this part is the most important skill in the unit.

---

# 1️⃣ WHAT IS PromQL? ⭐

> **PromQL** is a **functional query language** for selecting and aggregating time-series data in real time. The result of an expression can be shown as a graph, a table, or used in alert rules.

It is **read-only** (you can't modify data) and **not SQL**. Think of it as "spreadsheet formulas over time series".

Where to run queries: Prometheus UI (`:9090` → Query tab), Grafana panels, the HTTP API (`/api/v1/query`), `promtool query instant`.

## 1.1 The four data types ⭐⭐⭐

| Type | Meaning | Example |
|---|---|---|
| **Instant vector** | A set of series, each with **one sample at one timestamp** | `node_load1` |
| **Range vector** | A set of series, each with **a range of samples** over time | `node_load1[5m]` |
| **Scalar** | A single number | `3.14`, `time()` |
| **String** | Text (rarely used) | `"hello"` |

```
Instant vector:  node_load1                →  {instance="a"} 0.42    {instance="b"} 1.10      (one value per series, "now")
Range vector:    node_load1[2m]            →  {instance="a"} 0.40 0.41 0.42 0.43 0.42 ...     (many values per series)
```

⚠️ You **cannot graph a range vector directly**. Functions like `rate()` take a range vector and return an instant vector, which you can graph.

---

# 2️⃣ SELECTORS ⭐⭐⭐

## 2.1 Instant vector selectors

```promql
node_cpu_seconds_total                              # all series with this metric name
node_cpu_seconds_total{mode="idle"}                 # label matcher: exact match
node_cpu_seconds_total{mode="idle", instance="10.0.1.5:9100"}   # several matchers = AND
http_requests_total{status=~"5.."}                  # regex match: status codes 500-599
http_requests_total{status!~"2.."}                  # NOT matching regex
http_requests_total{method!="GET"}                  # not equal
{__name__=~"node_memory_.*_bytes"}                  # match many metric names by regex (use sparingly: heavy)
```

| Matcher | Meaning |
|---|---|
| `=` | Equals |
| `!=` | Not equals |
| `=~` | Regex matches (**fully anchored**: `"5.."` must match the whole value) |
| `!~` | Regex does not match |

Rules: a selector must include at least one matcher that **doesn't match the empty string**; label values are case-sensitive; regex is RE2.

## 2.2 Range vector selectors: `[duration]`

```promql
node_cpu_seconds_total{mode="idle"}[5m]      # last 5 minutes of samples for each series
```

Durations: `ms`, `s`, `m`, `h`, `d`, `w`, `y` (e.g., `90s`, `1h30m`, `7d`).

## 2.3 `offset` and `@` modifiers (time travel)

```promql
node_load1 offset 1h                          # value 1 hour ago
rate(http_requests_total[5m] offset 1d)       # yesterday's rate at this time of day
node_load1 @ 1732358400                       # value at a fixed Unix timestamp
```

Compare now vs same time yesterday (spot regressions):

```promql
sum(rate(http_requests_total[5m])) / sum(rate(http_requests_total[5m] offset 1d))
```

## 2.4 How evaluation works: instant vs range queries, step, staleness ⭐

| Concept | Meaning |
|---|---|
| **Instant query** | Evaluate the expression at **one** timestamp (the **Table** view; used by alert rules) |
| **Range query** | Evaluate at many timestamps between `start` and `end` every **step** (the **Graph** view, Grafana panels) |
| **Step / resolution** | Distance between evaluation points (e.g., 15 s) |
| **Staleness / lookback delta** | An instant vector selector returns the most recent sample **within the last 5 minutes**; if a target disappears, its series is marked **stale** and stops appearing |

---

# 3️⃣ OPERATORS ⭐⭐

## 3.1 Arithmetic: `+  -  *  /  %  ^`

```promql
node_memory_MemTotal_bytes / 1024 / 1024 / 1024                  # bytes → GiB
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes      # ratio 0-1 (two vectors with identical labels match one-to-one)
rate(node_network_receive_bytes_total[5m]) * 8                   # bytes/s → bits/s
```

Operators between a vector and a scalar apply to each sample. Between two vectors, samples are matched by **identical label sets** (excluding `__name__`).

## 3.2 Comparison: `==  !=  >  <  >=  <=` (filtering) ⭐

By default, comparisons **filter**: they keep only the series where the condition is true.

```promql
node_load1 > 4                                   # only the series with load above 4
up == 0                                           # only DOWN targets   ← the classic alert
node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.10     # filesystems with < 10% free
```

With `bool`, they return **0 or 1** for every series (no filtering):

```promql
up == bool 1                                      # 1 for up, 0 for down, for each target
```

## 3.3 Logical / set operators: `and`, `or`, `unless`

```promql
(node_load1 > 4) and (node_memory_MemAvailable_bytes < 1e9)   # both conditions on the same instance (matching labels)
up == 0 or node_load1 > 8                                       # union
(up == 1) unless (node_load1 > 8)                               # in A but not in B
```

## 3.4 Vector matching: `on`, `ignoring`, `group_left`, `group_right` ⭐

When label sets differ between the two sides, tell PromQL how to match:

```promql
# CPU load per core: node_load1 has labels {instance}, cpu count also by {instance}
node_load1 / on(instance) count by (instance) (node_cpu_seconds_total{mode="idle"})

# ignore a label that differs
rate(a[5m]) / ignoring(status) rate(b[5m])

# many-to-one: attach labels from the "one" side
sum by (instance, mode) (rate(node_cpu_seconds_total[5m]))
  / on(instance) group_left
sum by (instance) (rate(node_cpu_seconds_total[5m]))
```

| Modifier | Meaning |
|---|---|
| `on(l1,l2)` | Match only on these labels |
| `ignoring(l1)` | Match on all labels **except** these |
| `group_left` / `group_right` | Allow many-to-one / one-to-many matching and copy labels from the "one" side |

Symptom of a matching mistake: **empty result**. Fix with `on(...)`/`ignoring(...)`.

## 3.5 Precedence (high → low)

```
^                                  (right associative)
* / % atan2
+ -
== != <= < >= >
and unless
or
```

Use parentheses liberally.

---

# 4️⃣ AGGREGATION OPERATORS ⭐⭐⭐

Aggregations reduce many series into fewer.

| Operator | Meaning |
|---|---|
| `sum` | Total |
| `avg` | Average |
| `min`, `max` | Smallest / largest |
| `count` | Number of series |
| `count_values` | Count series per distinct value |
| `stddev`, `stdvar` | Spread |
| `topk(k, x)`, `bottomk(k, x)` | Highest / lowest k series (keeps labels) |
| `quantile(φ, x)` | φ-quantile across series (not the same as `histogram_quantile`) |
| `group` | Returns 1 for each group (existence) |

## 4.1 `by` and `without`

```promql
sum(rate(http_requests_total[5m]))                               # one number: total request rate
sum by (job) (rate(http_requests_total[5m]))                     # one series per job
sum by (job, status) (rate(http_requests_total[5m]))             # per job and status
sum without (instance, pod) (rate(http_requests_total[5m]))      # keep everything EXCEPT instance and pod
avg by (instance) (node_load1)
count by (job) (up)                                               # how many targets per job
count(up == 0)                                                    # how many targets are down
topk(3, sum by (instance) (rate(node_network_receive_bytes_total[5m])))
```

`by (a,b)` keeps only labels a,b. `without (a,b)` drops a,b and keeps the rest.

⭐ **Golden rule:** for counters, **`rate()` first, then `sum`** (never `rate(sum(...))`), so counter resets per instance are handled correctly.

---

# 5️⃣ FUNCTIONS ⭐⭐⭐

## 5.1 `rate`, `irate`, `increase`, `delta` (counters and gauges)

| Function | Input | Returns | Use |
|---|---|---|---|
| **`rate(v[range])`** | counter range vector | **Per-second average** increase over the window, handles resets, extrapolates edges | Graphs and alerts (the default) |
| **`irate(v[range])`** | counter | Per-second rate from the **last two samples** (instant) | Very spiky/fast-moving graphs; **not** for alerts (too noisy) |
| **`increase(v[range])`** | counter | **Total** increase over the window (= `rate × seconds`) | "How many in the last hour?" |
| **`delta(v[range])`** | **gauge** | Difference between first and last value in window | Gauge change |
| **`deriv(v[range])`** | gauge | Per-second derivative (linear regression) | Trend of a gauge |

```promql
rate(http_requests_total[5m])                  # requests/second (5-minute average)
increase(http_requests_total[1h])              # requests in the last hour (e.g., 108000)
irate(node_network_receive_bytes_total[2m])    # instantaneous bytes/s
delta(node_memory_MemAvailable_bytes[1h])      # change in memory over the last hour
```

### Choosing the range window ⭐

```
Window must contain AT LEAST 2 samples; rule of thumb: window ≥ 4 × scrape_interval
scrape_interval 15s → minimum [1m]; common [5m]
Short window = responsive but noisy ; long window = smooth but slow to react
In Grafana use $__rate_interval (auto-chosen from the panel resolution and scrape interval)
```

## 5.2 `*_over_time` functions (aggregate over a window per series)

```promql
avg_over_time(node_load1[1h])
max_over_time(node_memory_MemAvailable_bytes[1d])
min_over_time(up[10m])                    # was the target ever down in the last 10 minutes? (0 = yes)
quantile_over_time(0.95, node_load1[1h])
count_over_time(up[1h])
```

## 5.3 Prediction and trend

```promql
# will the filesystem be FULL in 4 hours, based on the last 6 hours of data?
predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h], 4 * 3600) < 0
```

Excellent for **proactive alerts**: warn *before* the disk fills.

## 5.4 Existence and change detection

```promql
absent(up{job="flask-app"})                    # returns 1 if NO such series exists (job vanished entirely)
absent_over_time(up{job="node"}[10m])          # absent for the whole 10 minutes
changes(node_boot_time_seconds[1d])            # how many times the value changed (reboots!)
resets(http_requests_total[1h])                # how many counter resets (restarts)
```

## 5.5 Math, rounding, clamping, sorting

```promql
round(x, 0.01)        ceil(x)        floor(x)        abs(x)        sqrt(x)        ln(x)        exp(x)
clamp_min(x, 0)       clamp_max(x, 100)       clamp(x, 0, 100)
sort(x)               sort_desc(x)            # for table views
vector(0)             scalar(x)               # convert between scalar and vector
```

Tip: provide a default when a series may be missing:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) or vector(0)
```

## 5.6 Time functions

```promql
time()                                          # current Unix time (scalar)
time() - node_boot_time_seconds                 # uptime in seconds
timestamp(up)                                   # sample timestamp
hour(), minute(), day_of_week(), month()        # calendar functions (UTC)
# restrict an alert to business hours (Mon-Fri, 09:00-17:59 UTC) by AND-ing a condition:
<your_alert_expression> and on() (day_of_week() > 0 and day_of_week() < 6 and hour() >= 9 and hour() < 18)
```

## 5.7 Label functions

```promql
label_replace(up, "host", "$1", "instance", "(.*):.*")      # host = instance without the port
label_join(up, "id", "-", "job", "instance")                # id = job-instance
```

---

# 6️⃣ HISTOGRAMS AND PERCENTILES ⭐⭐⭐

Given a histogram `myapp_http_request_duration_seconds` (Part 2):

## 6.1 Percentiles with `histogram_quantile`

```promql
# 95th percentile latency across ALL instances, per endpoint (5-minute window)
histogram_quantile(
  0.95,
  sum by (le, endpoint) (rate(myapp_http_request_duration_seconds_bucket[5m]))
)

# overall p99
histogram_quantile(0.99, sum by (le) (rate(myapp_http_request_duration_seconds_bucket[5m])))
```

Rules:

```
1. rate() the _bucket counters first (percentiles of "recent" behavior, not since process start)
2. sum by (le, <other labels you want to keep>)   ← always keep `le`
3. Result is an ESTIMATE (linear interpolation inside a bucket): accuracy depends on bucket boundaries
4. Never average percentiles across instances (avg(p95) is wrong); aggregate buckets, then compute the quantile
```

## 6.2 Average latency

```promql
sum(rate(myapp_http_request_duration_seconds_sum[5m])) / sum(rate(myapp_http_request_duration_seconds_count[5m]))
```

## 6.3 SLI: fraction of requests faster than 300 ms ⭐ (needs a bucket at `le="0.25"` or `le="0.5"`; use the bucket that matches your threshold)

```promql
sum(rate(myapp_http_request_duration_seconds_bucket{le="0.5"}[5m]))
/
sum(rate(myapp_http_request_duration_seconds_count[5m]))
```

Result 0.97 = 97% of requests were served within 500 ms.

## 6.4 Apdex-style score (satisfied ≤ T, tolerating ≤ 4T)

```promql
(
  sum(rate(myapp_http_request_duration_seconds_bucket{le="0.25"}[5m]))
  + sum(rate(myapp_http_request_duration_seconds_bucket{le="1"}[5m]))
) / 2
/ sum(rate(myapp_http_request_duration_seconds_count[5m]))
```

(Here T = 0.25 s, tolerating up to 1 s.)

---

# 7️⃣ SUBQUERIES ⭐

A **subquery** runs an expression over a range at a given resolution, producing a range vector you can feed to `*_over_time`:

```promql
# highest 5-minute request rate observed in the last hour, evaluated every minute
max_over_time( sum(rate(http_requests_total[5m]))[1h:1m] )

# the worst CPU usage % in the last 24 hours
max_over_time( (100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)[24h:5m] )
```

Syntax: `<expression>[<range>:<resolution>]`. Subqueries are expensive; for repeated use, create a **recording rule**.

---

# 8️⃣ NODE EXPORTER QUERY COOKBOOK 🖥️⭐⭐⭐ (PRACTICALS 8 AND 9)

Run these in the Prometheus UI and use them as Grafana panels. (`instance` is the label identifying each host.)

## 8.1 Availability

```promql
up{job="node"}                                         # 1 = healthy target, 0 = down
count(up{job="node"} == 0)                             # number of nodes down
```

## 8.2 CPU

```promql
# CPU usage % per instance (100 - idle%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# CPU usage % per core
100 - (avg by (instance, cpu) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# CPU time by mode (% of one average core): user, system, iowait, steal...
avg by (instance, mode) (rate(node_cpu_seconds_total{mode!="idle"}[5m])) * 100

# I/O wait % (CPU waiting on disk)
avg by (instance) (rate(node_cpu_seconds_total{mode="iowait"}[5m])) * 100

# steal % (noisy neighbour on shared cloud hardware, e.g., burstable t-instances)
avg by (instance) (rate(node_cpu_seconds_total{mode="steal"}[5m])) * 100

# number of CPU cores
count by (instance) (node_cpu_seconds_total{mode="idle"})
```

## 8.3 Load average (saturation)

```promql
node_load1                                             # 1-minute load
node_load5
# load per core (≥ 1 sustained means CPU saturation)
node_load1 / on(instance) count by (instance) (node_cpu_seconds_total{mode="idle"})
```

⚠️ The `on(instance)` is **required**: `node_load1` carries labels like `job`, `env`, `role`, while `count by (instance)` keeps only `instance`. Without `on(instance)` the label sets differ and the result is **empty** (a classic silent failure, verified on live data).

## 8.4 Memory

```promql
# memory used % (based on MemAvailable)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# memory used in GiB
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / 1024 / 1024 / 1024

# available memory in MiB
node_memory_MemAvailable_bytes / 1024 / 1024

# memory breakdown (stacked graph)
node_memory_MemTotal_bytes - node_memory_MemFree_bytes - node_memory_Buffers_bytes - node_memory_Cached_bytes   # "used" (apps)
node_memory_Buffers_bytes + node_memory_Cached_bytes                                                            # buffers + page cache

# swap used % (division by zero if the host has no swap → ignore NaN)
(1 - node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes) * 100
```

Why MemAvailable? Linux uses spare RAM as cache; **MemFree** looks scary-low but is not a problem. **MemAvailable** estimates how much memory is truly usable.

## 8.5 Disk space (storage)

```promql
# filesystem used % (exclude virtual filesystems)
100 - (
  node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs"}
  / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs"} * 100
)

# free space in GiB per mountpoint
node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs"} / 1024 / 1024 / 1024

# inode usage % (disk "full" of files even if space remains)
100 - (node_filesystem_files_free{fstype!~"tmpfs|overlay"} / node_filesystem_files{fstype!~"tmpfs|overlay"} * 100)

# will the root filesystem fill within 24 hours? (alert candidate)
predict_linear(node_filesystem_avail_bytes{mountpoint="/", fstype!~"tmpfs|overlay"}[6h], 24 * 3600) < 0
```

## 8.6 Disk I/O

```promql
rate(node_disk_read_bytes_total[5m])                   # read throughput bytes/s
rate(node_disk_written_bytes_total[5m])                # write throughput bytes/s
rate(node_disk_reads_completed_total[5m])              # read IOPS
rate(node_disk_writes_completed_total[5m])             # write IOPS
rate(node_disk_io_time_seconds_total[5m]) * 100        # disk utilization % (≈100% = saturated)
```

## 8.7 Network

```promql
# inbound / outbound traffic in bits per second (exclude loopback)
rate(node_network_receive_bytes_total{device!="lo"}[5m]) * 8
rate(node_network_transmit_bytes_total{device!="lo"}[5m]) * 8

# errors and drops per second
rate(node_network_receive_errs_total[5m]) + rate(node_network_transmit_errs_total[5m])
rate(node_network_receive_drop_total[5m]) + rate(node_network_transmit_drop_total[5m])

# total traffic across all interfaces of a host
sum by (instance) (rate(node_network_receive_bytes_total{device!="lo"}[5m]))

# established TCP connections
node_netstat_Tcp_CurrEstab
```

## 8.8 System

```promql
time() - node_boot_time_seconds                        # uptime in seconds
(time() - node_boot_time_seconds) / 86400              # uptime in days
node_filefd_allocated / node_filefd_maximum * 100      # file-descriptor usage %
rate(node_context_switches_total[5m])                  # context switches per second
node_uname_info                                        # OS/kernel info (labels: nodename, release, machine)
abs(node_timex_offset_seconds)                         # clock drift (time sync)
```

## 8.9 Fleet views

```promql
topk(5, 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))     # 5 busiest hosts
bottomk(3, node_filesystem_avail_bytes{mountpoint="/"})                                        # 3 hosts with least free space
count(node_uname_info)                                                                         # how many hosts report
```

---

# 9️⃣ RED QUERIES FOR YOUR FLASK APP 📈

Using the metrics from Part 2 (`myapp_http_requests_total`, `myapp_http_request_duration_seconds`, `myapp_http_requests_in_progress`, `myapp_orders_created_total`):

```promql
# RATE: requests per second, total and per endpoint
sum(rate(myapp_http_requests_total[1m]))
sum by (endpoint) (rate(myapp_http_requests_total[1m]))

# ERRORS: 5xx per second and as a ratio (error rate %)
sum(rate(myapp_http_requests_total{status=~"5.."}[1m]))
100 * sum(rate(myapp_http_requests_total{status=~"5.."}[1m])) / sum(rate(myapp_http_requests_total[1m]))

# per endpoint error ratio (only endpoints with traffic)
sum by (endpoint) (rate(myapp_http_requests_total{status=~"5.."}[5m]))
  / sum by (endpoint) (rate(myapp_http_requests_total[5m]))

# DURATION: p50, p95, p99 and average
histogram_quantile(0.50, sum by (le) (rate(myapp_http_request_duration_seconds_bucket[5m])))
histogram_quantile(0.95, sum by (le) (rate(myapp_http_request_duration_seconds_bucket[5m])))
histogram_quantile(0.99, sum by (le) (rate(myapp_http_request_duration_seconds_bucket[5m])))
sum(rate(myapp_http_request_duration_seconds_sum[5m])) / sum(rate(myapp_http_request_duration_seconds_count[5m]))

# SATURATION-ish: requests in flight
sum(myapp_http_requests_in_progress)

# BUSINESS: orders in the last hour and per minute
increase(myapp_orders_created_total[1h])
rate(myapp_orders_created_total[5m]) * 60
```

---

# 🔟 RECORDING RULES ⭐⭐

> A **recording rule** evaluates a PromQL expression on a schedule and **stores the result as a new time series**. Use them to **precompute expensive or frequently used queries** (dashboards load faster, alerts become simpler, federation/remote-write sends fewer series).

## 10.1 Naming convention

```
level:metric:operations
  level      = aggregation labels kept (e.g., instance, job)
  metric     = base metric name
  operations = functions applied, newest first (rate5m, ratio)

instance:node_cpu_utilisation:rate5m
job:myapp_http_requests:rate5m
job:myapp_http_request_error_ratio:rate5m
```

## 10.2 Rule file `rules/recording.yml`

```yaml
groups:
  - name: node_recording
    interval: 15s                     # optional; default = global evaluation_interval
    rules:
      - record: instance:node_cpu_utilisation:rate5m
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

      - record: instance:node_memory_utilisation:ratio
        expr: 1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

      - record: instance:node_network_receive_bits:rate5m
        expr: sum by (instance) (rate(node_network_receive_bytes_total{device!="lo"}[5m])) * 8

  - name: app_recording
    rules:
      - record: job:myapp_http_requests:rate5m
        expr: sum by (job) (rate(myapp_http_requests_total[5m]))
      - record: job:myapp_http_request_error_ratio:rate5m
        expr: |
          sum by (job) (rate(myapp_http_requests_total{status=~"5.."}[5m]))
          / sum by (job) (rate(myapp_http_requests_total[5m]))
      - record: job:myapp_http_request_duration_seconds:p95_5m
        expr: histogram_quantile(0.95, sum by (job, le) (rate(myapp_http_request_duration_seconds_bucket[5m])))
```

Enable in `prometheus.yml`:

```yaml
rule_files:
  - "rules/*.yml"
```

Validate and reload:

```bash
promtool check rules rules/recording.yml
curl -X POST http://localhost:9090/-/reload
```

Then query the cheap precomputed series directly: `instance:node_cpu_utilisation:rate5m`.

## 10.3 When to use recording rules

```
✔ Dashboard panels that run heavy aggregations over many series or long ranges
✔ Queries reused by many alerts/dashboards (single source of truth)
✔ SLO calculations over long windows (30 days): compute short-window ratios in recording rules, then aggregate them
✔ Before federating/remote-writing: send aggregates, not raw high-cardinality series (Part 6)
✘ Don't pre-record everything: each rule adds a stored series and evaluation cost
```

---

# 1️⃣1️⃣ SLO AND AVAILABILITY QUERIES ⭐

```promql
# availability SLI over 5 minutes: fraction of requests that are NOT 5xx
1 - (
  sum(rate(myapp_http_requests_total{status=~"5.."}[5m]))
  / sum(rate(myapp_http_requests_total[5m]))
)

# error budget burn rate against a 99.9% SLO (burn rate 1 = spending exactly the budget; >1 = faster than allowed)
(
  sum(rate(myapp_http_requests_total{status=~"5.."}[1h]))
  / sum(rate(myapp_http_requests_total[1h]))
) / (1 - 0.999)
```

Burn-rate alerts (multi-window) are in Part 4. Evaluating `[30d]` ranges directly is heavy; use recording rules.

---

# 1️⃣2️⃣ MONITORING PROMETHEUS ITSELF 🪞

```promql
prometheus_tsdb_head_series                                  # active series (memory driver)
rate(prometheus_tsdb_head_samples_appended_total[5m])        # ingestion rate (samples/s)
scrape_duration_seconds{job="node"}                          # slow scrapes
sum by (job) (scrape_samples_scraped)                        # samples per job (find big producers)
rate(prometheus_rule_evaluation_failures_total[5m])          # failing rules
prometheus_tsdb_storage_blocks_bytes                         # disk used by blocks
prometheus_notifications_dropped_total                       # alerts failing to reach Alertmanager
topk(10, count by (__name__) ({__name__=~".+"}))             # top metrics by series count (heavy; prefer TSDB Status page)
```

"Who watches the watcher": alert on these and run an external/second Prometheus for meta-monitoring (Part 6).

---

# 1️⃣3️⃣ COMMON MISTAKES AND DEBUGGING METHOD 🚨🛠️

| Mistake | Symptom | Fix |
|---|---|---|
| `rate()` on a gauge | Nonsense values | Use rate only on counters; gauges directly or `deriv/delta` |
| Graphing a raw counter | Ever-rising line | Use `rate()` / `increase()` |
| `sum(rate(...))` written as `rate(sum(...))` | Wrong results on restarts | `sum(rate(...))` |
| Range window too small | "No data"/jumpy | `[≥ 4× scrape_interval]` |
| Averaging percentiles (`avg(histogram_quantile(...))`) | Statistically wrong | Aggregate `_bucket` by `le` first |
| Forgetting `le` in `sum by` for histograms | `histogram_quantile` returns NaN | `sum by (le, ...)` |
| Binary operation returns nothing | Label sets differ | `on(...)`/`ignoring(...)`, check labels in Table view |
| Division by zero/NaN | Gaps in graphs | Guard: `... > 0`, `clamp_min`, or `or vector(0)` |
| Regex without anchoring thought | `status=~"5"` matches only "5" | Regex is fully anchored: use `5..` |
| Using `{__name__=~...}` everywhere | Slow, heavy queries | Query specific metric names |
| Counter reset in `increase()` | Slight under-count | Normal; use rate/increase (they handle resets) |
| Expecting exact integers from `increase()` | Fractions (e.g., 7.98) | Extrapolation; round if needed |

## Debugging method ✅

```
1. Start with the bare metric:           node_cpu_seconds_total                    (does the series exist? what labels?)
2. Add selectors:                         node_cpu_seconds_total{mode="idle"}
3. Wrap in rate():                        rate(node_cpu_seconds_total{mode="idle"}[5m])
4. Aggregate step by step:                avg by (instance) (...)
5. Do the maths last:                     100 - (... * 100)
6. Use the TABLE view to inspect labels/values; use `count(...)` to verify how many series you expect
7. Check Status → Targets (is the data even arriving?) and the time range/step
```

---

# 1️⃣4️⃣ PRACTICE QUESTIONS (WITH ANSWERS) 🧪

**1. List all targets that are down.** `up == 0`

**2. How many targets per job are up?** `count by (job) (up == 1)`

**3. CPU usage % per host over 5 minutes.** `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`

**4. Memory used % per host.** `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100`

**5. Root filesystem used %.** `100 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100`

**6. Inbound network traffic in Mbit/s for `eth0`/`ens5`-type devices (exclude loopback).** `rate(node_network_receive_bytes_total{device!="lo"}[5m]) * 8 / 1e6`

**7. Which host will run out of disk (`/`) within 6 hours?** `predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[3h], 6*3600) < 0`

**8. Error percentage of the Flask app.** `100 * sum(rate(myapp_http_requests_total{status=~"5.."}[5m])) / sum(rate(myapp_http_requests_total[5m]))`

**9. p95 latency per endpoint.** `histogram_quantile(0.95, sum by (le, endpoint) (rate(myapp_http_request_duration_seconds_bucket[5m])))`

**10. Orders created in the last 24 hours.** `increase(myapp_orders_created_total[24h])`

**11. Uptime of each host in hours.** `(time() - node_boot_time_seconds) / 3600`

**12. Top 3 hosts by CPU usage.** `topk(3, 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))`

---

# 1️⃣5️⃣ VIVA QUESTIONS (PART 3) 🎯

**Q1. What is PromQL and which data types does it have?** Prometheus's functional query language; instant vector, range vector, scalar, string.

**Q2. Instant vector vs range vector?** An instant vector has one sample per series at a point in time; a range vector has multiple samples over a window (`metric[5m]`).

**Q3. What do `rate`, `irate` and `increase` do?** `rate`: per-second average over a window; `irate`: per-second from the last two samples (spiky); `increase`: total increase over the window.

**Q4. Why use `rate()` on counters?** Counters only increase; rate shows speed of change and handles resets.

**Q5. `sum by` vs `sum without`?** `by` keeps only listed labels; `without` drops listed labels and keeps the rest.

**Q6. How do you compute CPU usage % from Node Exporter?** `100 - avg by (instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100`.

**Q7. Why MemAvailable rather than MemFree?** MemAvailable accounts for reclaimable cache, giving the memory actually usable by applications.

**Q8. How do you calculate the 95th percentile latency?** `histogram_quantile(0.95, sum by (le)(rate(..._bucket[5m])))`.

**Q9. Why must `le` be kept in the aggregation?** `histogram_quantile` needs the bucket boundaries to interpolate.

**Q10. What is `predict_linear` used for?** Forecasting a gauge using linear regression, e.g., predicting disk full time.

**Q11. What do `on()` and `group_left` do?** `on()` restricts matching labels between two vectors; `group_left` allows many-to-one matching and copies labels from the one side.

**Q12. What is a recording rule and why use it?** A rule that precomputes a query and stores it as a new series, speeding dashboards and simplifying alerts.

**Q13. What does `absent()` do?** Returns 1 when the vector has no series, used to detect a vanished target.

**Q14. Why is `avg(histogram_quantile(...))` wrong?** Percentiles can't be averaged; aggregate buckets then compute the quantile.

**Q15. What window size should you use for `rate()`?** At least four scrape intervals (e.g., `[1m]` for 15 s scrapes; commonly `[5m]`).

---

# 🧠 PART 3 MASTER SHEET

```
TYPES: instant vector | range vector (metric[5m]) | scalar | string ; instant query (table, alerts) vs range query (graph, step) ; staleness 5m
SELECTORS: {label="x"} = != =~ !~ (regex anchored) ; offset 1h ; @ timestamp
OPERATORS: + - * / % ^ ; comparison filters (use `bool` to return 0/1) ; and / or / unless ; on() ignoring() group_left/right ; use parentheses
AGGREGATION: sum avg min max count stddev topk bottomk quantile group ; by(...) keep | without(...) drop ; rate FIRST then sum
FUNCTIONS: rate (default) | irate (spiky, not alerts) | increase | delta | deriv | predict_linear | *_over_time | absent | changes | resets
           clamp, round, sort_desc, time(), label_replace ; window ≥ 4 × scrape_interval ; Grafana $__rate_interval
HISTOGRAM: histogram_quantile(0.95, sum by (le, ...) (rate(x_bucket[5m]))) ; avg = rate(_sum)/rate(_count) ; never avg percentiles
NODE CHEATS: CPU% = 100 - avg(rate(idle))*100 | MEM% = (1 - MemAvailable/MemTotal)*100 | DISK% = 100 - avail/size*100
             NET = rate(bytes_total{device!="lo"})*8 | UPTIME = time() - node_boot_time_seconds | DISK FULL = predict_linear(avail[6h], 4*3600) < 0
RED: rate(requests_total) | error ratio = 5xx rate / total rate | p95 = histogram_quantile
RECORDING RULES: level:metric:operations ; precompute expensive queries ; promtool check rules
DEBUG: bare metric → selectors → rate → aggregate → math ; Table view ; check Targets ; mind label mismatches (on/ignoring)
```

---

## 🫡 PART 3 STATUS

You can now write PromQL for selection, rates, aggregation, histograms, prediction and recording rules, and you have a complete Node Exporter and Flask-app query cookbook for the practicals.

**STOP here.** Next up:

> **PART 4 — Alerting (alert rules, `for`, Alertmanager routing/grouping/inhibition/silencing, receivers, SLO burn-rate alerts, testing with promtool) 🚨**
