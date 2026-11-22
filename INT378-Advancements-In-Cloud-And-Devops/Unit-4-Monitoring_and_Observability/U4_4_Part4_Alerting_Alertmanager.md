# 🚨 UNIT IV — PART 4: ALERTING (PROMETHEUS RULES + ALERTMANAGER)

> **Covered in this document:** how alerting is split between Prometheus (decides *when*) and Alertmanager (decides *who and how*) → **alert rule anatomy** (`alert`, `expr`, `for`, `keep_firing_for`, `labels`, `annotations`, templating) → the alert **lifecycle** (inactive → pending → firing → resolved) → a **complete alert rule library** for nodes and the Flask app (InstanceDown, HighCPU, HighMemory, DiskSpaceLow, DiskWillFill, HighLoad, HighErrorRate, HighLatency, Watchdog) → validating and **unit-testing rules with `promtool`** → **Alertmanager architecture** (receive, deduplicate, group, route, inhibit, silence, notify, HA) → `alertmanager.yml` in full (global, route tree, timings, matchers, `continue`, receivers: email, Slack, webhook, PagerDuty, SNS) → **inhibition rules** and **silences** → **time intervals** (maintenance windows, office hours) → notification **templates** → a **webhook receiver** script you can run in the lab → `amtool` → **SLO burn-rate alerting** (multi-window, multi-burn-rate) with full rules → alert design best practices and anti-patterns → troubleshooting table → viva → master sheet.

---

Absolutely mate. 😤🔥 Part 3 taught queries. Part 4 turns **queries into notifications**: this is the "define alerting rules and validate alert generation" half of Practical 8.

---

# 1️⃣ HOW ALERTING WORKS: TWO PROGRAMS, TWO JOBS ⭐⭐⭐

```
 ┌───────────────────────── PROMETHEUS ─────────────────────────┐          ┌──────────────── ALERTMANAGER ────────────────┐
 │ evaluates ALERT RULES every evaluation_interval:              │  HTTP    │ 1. receive alerts (dedupe)                    │   Email
 │   expr true? → pending (wait `for:`) → FIRING                  │ ───────▶ │ 2. group related alerts                       │ ─▶ Slack
 │ sends firing alerts (and resolved) repeatedly to Alertmanager │ alerts   │ 3. route by labels to the right receiver      │   PagerDuty
 └──────────────────────────────────────────────────────────────┘          │ 4. inhibit (suppress) / silence (mute)        │   Webhook
                                                                             │ 5. notify, repeat, send "resolved"            │   SNS...
   "WHEN is something wrong?"                                                └──────────────────────────────────────────────┘
                                                                               "WHO should hear about it, how, and how often?"
```

| | Prometheus | Alertmanager |
|---|---|---|
| Responsibility | **Detection**: evaluate expressions, hold alert state | **Notification management**: dedupe, group, route, silence, send |
| Config | `rule_files` (alerting rules) | `alertmanager.yml` |
| Knows about | Metrics and thresholds | Teams, channels, schedules, maintenance windows |
| Port | 9090 | **9093** (cluster gossip 9094) |

Why separate? A single Prometheus detects, but **many** Prometheus servers can send to a **shared** Alertmanager that prevents notification storms (e.g., 200 pods down = **one** grouped notification, not 200 emails).

Also exists: **Grafana Alerting** (rules defined in Grafana, Part 5). Prometheus + Alertmanager is the classic, GitOps-friendly approach and is the focus of Practical 8.

---

# 2️⃣ ALERTING RULE ANATOMY ⭐⭐⭐

Rules live in YAML files referenced by `rule_files`, organized in **groups** (rules in a group are evaluated sequentially, at the group's interval).

```yaml
groups:
  - name: node_alerts                 # group name
    interval: 15s                     # optional; defaults to global evaluation_interval
    rules:
      - alert: InstanceDown                                  # alert name (CamelCase by convention)
        expr: up{job="node"} == 0                            # PromQL: returns series where the condition is TRUE
        for: 2m                                              # must stay true this long before FIRING (avoid flapping)
        keep_firing_for: 0s                                  # (optional) keep firing this long after expr turns false (reduce flapping)
        labels:                                              # extra labels attached to the alert (used by Alertmanager routing)
          severity: critical
          team: infra
        annotations:                                         # human-readable info (templated)
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.job }}/{{ $labels.instance }} has been unreachable for more than 2 minutes."
          runbook_url: "https://wiki.example.com/runbooks/instance-down"
```

| Field | Meaning |
|---|---|
| **`alert`** | The name; becomes label `alertname` |
| **`expr`** | A PromQL expression; **each returned series = one active alert** (labels come from the series) |
| **`for`** | **Pending duration**: how long the expression must be continuously true before the alert fires. `0s` = fire immediately |
| **`keep_firing_for`** | Keep the alert firing for this long after the condition clears (anti-flap) |
| **`labels`** | Static labels added to the alert (severity, team, env): **drive routing** |
| **`annotations`** | Descriptive text, links: `summary`, `description`, `runbook_url`, `dashboard_url` |

## 2.1 Templating in labels/annotations

Go templates are available:

| Variable | Meaning |
|---|---|
| `{{ $labels.<name> }}` | Label value of the alerting series (e.g., `$labels.instance`) |
| `{{ $value }}` | The numeric value of the expression at evaluation time |
| `{{ $externalLabels.<name> }}` | Prometheus `external_labels` |
| `{{ $value | humanize }}` | 1234567 → "1.235M" |
| `{{ $value | humanizePercentage }}` | 0.8734 → "87.34%" |
| `{{ $value | humanizeDuration }}` | 3725 → "1h 2m 5s" |
| `{{ $value | printf "%.1f" }}` | Format a number |

Example: `description: "CPU is {{ $value | printf \"%.1f\" }}% on {{ $labels.instance }}"`.

⚠️ Keep `{{ $value }}` **out of labels** (it changes every evaluation and would create a new alert each time); use it in **annotations** only.

## 2.2 The alert lifecycle ⭐⭐

```
                expr false                  expr true               expr true continuously for `for:`
   INACTIVE ─────────────────────▶ PENDING ──────────────────────────────────────────────▶ FIRING
      ▲                               │ expr becomes false (timer reset)                       │
      └───────────────────────────────┘◀───────────────────────────────────────────────────────┘ expr false → RESOLVED (notification "resolved")
```

| State | Meaning | Visible in |
|---|---|---|
| **Inactive** | Condition false | Alerts page (green) |
| **Pending** | Condition true but `for:` not yet satisfied | Yellow |
| **Firing** | Condition true for ≥ `for:`; sent to Alertmanager | Red |
| **Resolved** | Condition cleared; Alertmanager informs receivers | Notification |

Prometheus also exposes alert state as metrics: `ALERTS{alertname, alertstate="firing|pending", ...}` (use it in Grafana to show active alerts) and `ALERTS_FOR_STATE`.

Prometheus **re-sends firing alerts to Alertmanager every minute** (default); Alertmanager decides how often humans are notified (`repeat_interval`).

## 2.3 Choosing `for:` ⭐

```
Too short (0-30s)  → flapping, false alarms on transient spikes
Too long (30m)     → slow detection
Typical: availability 1-5m ; CPU/memory saturation 5-10m ; disk-fill prediction 15-30m ; error-rate 5m ; SLO burn-rate: handled by windows
```

---

# 3️⃣ A COMPLETE RULE LIBRARY (COPY INTO `rules/alerts.yml`) ⭐⭐⭐

```yaml
groups:
  # ---------------- availability ----------------
  - name: availability
    rules:
      - alert: Watchdog                       # ALWAYS firing: proves the alerting pipeline works (dead man's switch)
        expr: vector(1)
        labels:
          severity: none
        annotations:
          summary: "Watchdog: alerting pipeline is alive"

      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.job }}/{{ $labels.instance }} has been unreachable for more than 2 minutes."

      - alert: TargetMissing                  # a whole job vanished (e.g., misconfigured discovery)
        expr: absent(up{job="node"})
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "No 'node' targets are being scraped"

  # ---------------- infrastructure (USE method) ----------------
  - name: node_resources
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | printf \"%.1f\" }}% (threshold 80%) for 5 minutes."

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | printf \"%.1f\" }}% (threshold 85%)."

      - alert: DiskSpaceLow
        expr: |
          100 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs"}
                 / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs"} * 100) > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disk almost full on {{ $labels.instance }} ({{ $labels.mountpoint }})"
          description: "{{ $labels.mountpoint }} is {{ $value | printf \"%.1f\" }}% full."

      - alert: DiskWillFillIn4Hours             # proactive: predicts running out of space
        expr: predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs"}[1h], 4 * 3600) < 0
        for: 30m
        labels:
          severity: critical
        annotations:
          summary: "Disk on {{ $labels.instance }} ({{ $labels.mountpoint }}) predicted to fill within 4 hours"

      - alert: HighLoadPerCore
        expr: node_load5 / count by (instance) (node_cpu_seconds_total{mode="idle"}) > 1.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High load on {{ $labels.instance }}"
          description: "5-minute load per core is {{ $value | printf \"%.2f\" }}."

      - alert: HighNetworkTraffic
        expr: sum by (instance) (rate(node_network_receive_bytes_total{device!="lo"}[5m])) * 8 > 400e6
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Inbound traffic above 400 Mbit/s on {{ $labels.instance }}"

  # ---------------- application (RED method) ----------------
  - name: flask_app
    rules:
      - alert: HighErrorRate
        expr: |
          sum by (job) (rate(myapp_http_requests_total{status=~"5.."}[5m]))
          / sum by (job) (rate(myapp_http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.job }}: more than 5% of requests are failing"
          description: "Error ratio is {{ $value | humanizePercentage }}."

      - alert: HighLatencyP95
        expr: |
          histogram_quantile(0.95, sum by (job, le) (rate(myapp_http_request_duration_seconds_bucket[5m]))) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.job }}: p95 latency above 500 ms"
          description: "p95 latency is {{ $value | humanizeDuration }}."

      - alert: NoTraffic                         # sudden silence can mean a broken route or upstream
        expr: sum(rate(myapp_http_requests_total[10m])) == 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "The app has received no requests for 15 minutes"
```

### Lab-friendly thresholds ⚠️

For **Practical 8** you want alerts to fire in minutes, not hours. Use temporary versions, for example **`HighCPUUsage` at `> 50` with `for: 1m`** while testing with a CPU stress tool, and an easy-to-trigger `InstanceDown` with `for: 30s`. Restore production-style values afterwards. (Part 8 gives the exact lab rules.)

## 3.1 Validate and load

```bash
promtool check rules rules/alerts.yml          # SUCCESS: 12 rules found
promtool check config prometheus.yml           # also checks that rule files exist/parse
curl -X POST http://localhost:9090/-/reload    # (needs --web.enable-lifecycle) or systemctl reload prometheus
```

Verify in the UI: **Alerts** (all rules and their state) and **Status → Rules** (last evaluation, duration, errors).

---

# 4️⃣ UNIT-TESTING ALERT RULES WITH `promtool` ⭐ (BE A PROFESSIONAL)

Test rules **before** production by feeding synthetic series:

`tests/alerts_test.yml`:

```yaml
rule_files:
  - ../rules/alerts.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'up{job="node", instance="a:9100"}'
        values: '1 1 0 0 0 0 0 0'          # up for 2 minutes, then down
    alert_rule_test:
      - eval_time: 1m                       # still up → no alert
        alertname: InstanceDown
        exp_alerts: []
      - eval_time: 5m                       # down since minute 2; `for: 2m` satisfied → firing
        alertname: InstanceDown
        exp_alerts:
          - exp_labels:
              severity: critical
              job: node
              instance: "a:9100"
            exp_annotations:
              summary: "Instance a:9100 is down"
              description: "node/a:9100 has been unreachable for more than 2 minutes."
```

Run:

```bash
promtool test rules tests/alerts_test.yml
# Unit Testing:  tests/alerts_test.yml
#   SUCCESS
```

Benefits: catches typos, wrong thresholds, wrong `for`, missing labels; runs in CI (Unit III pipeline!). You can also unit-test recording rules with `promql_expr_test`.

---

# 5️⃣ ALERTMANAGER ARCHITECTURE ⭐⭐⭐

```
 Prometheus #1 ──┐                         ┌──────────────────────────── ALERTMANAGER ─────────────────────────────┐
 Prometheus #2 ──┼─ POST /api/v2/alerts ─▶ │ Dispatcher → ROUTING TREE (match labels) → per-route GROUPS (buffer) │
 (HA pair)     ──┘                         │      ▲ inhibitor (suppress) ▲ silencer (mute)                         │
                                           │ Notification pipeline → retry → RECEIVERS (email/Slack/PagerDuty/…)  │
                                           │ Cluster (gossip :9094) shares silences & notification log with peers  │
                                           └───────────────────────────────────────────────────────────────────────┘
```

| Feature | What it does | Why it matters |
|---|---|---|
| **Deduplication** | Identical alerts from several Prometheus servers (HA) are merged | No double notifications |
| **Grouping** | Combines related alerts into **one** notification (e.g., `group_by: [alertname, cluster]`) | 100 pod alerts → 1 message |
| **Routing** | A tree matches alert labels to **receivers** | Critical → pager; warning → Slack; team labels → team channels |
| **Inhibition** | Suppress some alerts while another (more important) alert is firing | Node down → don't also page for its CPU/disk/service alerts |
| **Silencing** | Temporarily mute alerts by label matchers (UI or `amtool`), with an expiry | Planned maintenance |
| **Notification** | Sends to receivers with retries; sends **resolved** messages | Close the loop |
| **High availability** | Run 2-3 Alertmanagers in a gossip cluster; point **every** Prometheus at **all** of them (do not load-balance them) | No single point of failure |

---

# 6️⃣ `alertmanager.yml` IN FULL ⭐⭐⭐

```yaml
global:
  resolve_timeout: 5m                       # mark alerts resolved if Prometheus stops sending them
  # --- email (SMTP) defaults ---
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password_file: '/etc/alertmanager/smtp_password'   # keep secrets in files, not in the config
  smtp_require_tls: true

# ---------- routing tree ----------
route:
  receiver: team-email                      # default receiver (root route)
  group_by: ['alertname', 'instance']       # alerts with the same values are grouped into one notification
  group_wait: 30s                           # wait before sending the FIRST notification for a new group (collect related alerts)
  group_interval: 5m                        # wait before sending notifications about NEW alerts added to an existing group
  repeat_interval: 4h                       # re-send if still firing (reminder)

  routes:
    - matchers: [ 'alertname="Watchdog"' ]  # swallow the always-firing heartbeat (or route it to a deadman service)
      receiver: 'null'

    - matchers: [ 'severity="critical"' ]
      receiver: oncall
      repeat_interval: 1h
      continue: false                       # stop after this route (true = also evaluate following sibling routes)

    - matchers: [ 'severity="warning"' ]
      receiver: slack-warnings
      group_by: ['alertname']

    - matchers: [ 'team="database"' ]       # route by team label
      receiver: db-team

# ---------- inhibition ----------
inhibit_rules:
  - source_matchers: [ 'alertname="InstanceDown"' ]               # if a node is down...
    target_matchers: [ 'alertname=~"HighCPUUsage|HighMemoryUsage|DiskSpaceLow|HighLoadPerCore"' ]   # ...mute these for the SAME instance
    equal: ['instance']

# ---------- receivers ----------
receivers:
  - name: 'null'                            # blackhole

  - name: team-email
    email_configs:
      - to: 'team@example.com'
        send_resolved: true

  - name: slack-warnings
    slack_configs:
      - api_url_file: '/etc/alertmanager/slack_webhook_url'     # the Slack Incoming Webhook URL kept in a file
        channel: '#alerts'
        send_resolved: true
        title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
        text: >-
          {{ range .Alerts }}• {{ .Annotations.summary }}
          {{ end }}

  - name: oncall
    # PagerDuty (Events API v2): routing key from your PagerDuty service integration
    pagerduty_configs:
      - routing_key_file: '/etc/alertmanager/pagerduty_key'
        send_resolved: true
    # also email critical alerts
    email_configs:
      - to: 'oncall@example.com'
        send_resolved: true

  - name: db-team
    webhook_configs:
      - url: 'http://localhost:5001/alerts'                      # any HTTP endpoint (chat bridge, ticket creator, your own script)
        send_resolved: true
```

## 6.1 The route tree ⭐

```
ROOT route (default receiver: team-email)  group_by [alertname, instance]
 ├─ alertname="Watchdog"      → null
 ├─ severity="critical"       → oncall        (repeat every 1h)
 ├─ severity="warning"        → slack-warnings
 └─ team="database"           → db-team
(An alert goes to the FIRST matching child route; if none matches, it uses the root receiver. `continue: true` lets it match more routes.)
```

| Field | Meaning |
|---|---|
| **`receiver`** | Who gets it |
| **`matchers`** | Label conditions (`=`, `!=`, `=~`, `!~`); the older `match`/`match_re` keys are deprecated |
| **`group_by`** | Labels defining a notification group; `['...']` groups by all labels (no grouping) |
| **`group_wait`** | Delay before the first notification of a group (default 30s) |
| **`group_interval`** | Delay before notifying about new alerts in an existing group (default 5m) |
| **`repeat_interval`** | Re-notify while still firing (default 4h) |
| **`continue`** | Keep matching later sibling routes |
| **`mute_time_intervals` / `active_time_intervals`** | Time-based muting/activating (Section 9) |

## 6.2 Timing walkthrough (DRY RUN) 🏃

```
10:00:00  Alert A (HighCPUUsage, instance=web1) arrives → new group {alertname=HighCPUUsage, instance=web1}
10:00:30  group_wait expires → ONE notification sent (contains A)
10:02:00  Alert B (HighCPUUsage, instance=web1 / another core) joins the SAME group
10:05:30  group_interval (5m after last notification) → notification sent including B
14:05:30  Still firing → repeat_interval (4h) → reminder sent
14:20:00  Condition clears → "RESOLVED" notification (if send_resolved: true)
```

## 6.3 Receivers cheat-sheet

| Receiver | Config key | Notes |
|---|---|---|
| Email | `email_configs` | SMTP settings in `global`; Gmail needs an **app password** (2-step verification) |
| Slack | `slack_configs` | Incoming webhook URL (`api_url`/`api_url_file`) |
| PagerDuty | `pagerduty_configs` | `routing_key` (Events API v2) |
| Opsgenie | `opsgenie_configs` | (check the product's current status) |
| Webhook | `webhook_configs` | JSON POST to any URL: bridge to Teams, Telegram, Jira, custom code |
| AWS SNS | `sns_configs` | Fan out to email/SMS/Lambda (Unit I/VI link) |
| Telegram / Discord / Microsoft Teams / WeChat / Pushover / VictorOps | respective blocks | Available in recent versions; check your Alertmanager version's docs |

Secrets: prefer `*_file` options (`api_url_file`, `smtp_auth_password_file`, `routing_key_file`) with strict file permissions (`chmod 600`, owned by the `alertmanager` user).

---

# 7️⃣ INHIBITION AND SILENCES ⭐⭐

## 7.1 Inhibition (automatic suppression)

> **Inhibition** mutes **target** alerts while a **source** alert is firing and both share the labels listed in `equal`.

```
Source: InstanceDown{instance="web1"} FIRING
Targets muted: HighCPUUsage{instance="web1"}, DiskSpaceLow{instance="web1"} ... (same instance) → you only hear "web1 is down"
```

Common patterns: cluster down inhibits node alerts; node down inhibits service alerts; `critical` inhibits `warning` for the same alertname (`equal: [alertname, instance]`).

## 7.2 Silences (manual, temporary muting)

```
Alertmanager UI :9093 → Silences → New Silence → matchers (alertname="DiskSpaceLow", instance="db1:9100") → duration 2h → comment "planned disk resize" → Create
```

CLI with `amtool`:

```bash
amtool --alertmanager.url=http://localhost:9093 alert                     # list active alerts
amtool silence add alertname=HighCPUUsage instance=web1:9100 --duration=2h --comment="load test" --author="you"
amtool silence query                                                       # list silences
amtool silence expire <silence-id>
amtool check-config /etc/alertmanager/alertmanager.yml                    # validate config
amtool config routes show --config.file=/etc/alertmanager/alertmanager.yml     # print the routing tree
amtool config routes test --config.file=/etc/alertmanager/alertmanager.yml severity=critical team=infra   # which receiver would get this?
```

| | Inhibition | Silence |
|---|---|---|
| Trigger | Automatic (another alert firing) | Manual (human) |
| Defined in | `alertmanager.yml` | UI/API/`amtool` (runtime) |
| Use | Reduce redundant alerts | Maintenance windows, known issues |
| Expiry | When source resolves | Fixed end time |

---

# 8️⃣ TEMPLATES (CUSTOM NOTIFICATION TEXT) 📝

Go templating works in receiver fields. Common data:

```
{{ .Status }}                       firing | resolved
{{ .Alerts.Firing | len }}          number of firing alerts
{{ .GroupLabels.alertname }}        labels used in group_by
{{ .CommonLabels.severity }}        labels shared by all alerts in the group
{{ .CommonAnnotations.summary }}
{{ range .Alerts }} ... {{ .Labels.instance }} {{ .Annotations.description }} {{ .StartsAt }} {{ .GeneratorURL }} {{ end }}
```

Reusable template file `/etc/alertmanager/templates/custom.tmpl` + `templates: ['/etc/alertmanager/templates/*.tmpl']` in the config:

```
{{ define "slack.title" }}[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}{{ end }}
{{ define "slack.text" }}{{ range .Alerts }}*{{ .Labels.severity | toUpper }}* {{ .Annotations.summary }}
{{ if .Annotations.runbook_url }}Runbook: {{ .Annotations.runbook_url }}{{ end }}
{{ end }}{{ end }}
```

Use: `title: '{{ template "slack.title" . }}'`.

---

# 9️⃣ TIME INTERVALS: MAINTENANCE WINDOWS AND OFFICE HOURS ⏰

```yaml
time_intervals:
  - name: weekend
    time_intervals:
      - weekdays: ['saturday', 'sunday']
  - name: business-hours
    time_intervals:
      - weekdays: ['monday:friday']
        times:
          - start_time: '09:00'
            end_time: '18:00'
        location: 'Asia/Kolkata'

route:
  routes:
    - matchers: [ 'severity="warning"' ]
      receiver: slack-warnings
      active_time_intervals: ['business-hours']      # only notify during office hours
    - matchers: [ 'severity="info"' ]
      receiver: slack-warnings
      mute_time_intervals: ['weekend']               # never on weekends
```

(Older configs used `mute_time_intervals` at the top level; use the structure supported by your Alertmanager version and validate with `amtool check-config`.)

---

# 🔟 A WEBHOOK RECEIVER FOR THE LAB (NO EMAIL NEEDED) 🪝⭐

To **prove** alerts are generated and delivered without setting up SMTP, run a tiny receiver that prints whatever Alertmanager POSTs.

`webhook_receiver.py`:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import json
from datetime import datetime


class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        length = int(self.headers.get("Content-Length", 0))
        payload = json.loads(self.rfile.read(length) or b"{}")
        print("\n" + "=" * 70)
        print(datetime.now().isoformat(timespec="seconds"),
              "| status:", payload.get("status"),
              "| receiver:", payload.get("receiver"))
        for alert in payload.get("alerts", []):
            labels = alert.get("labels", {})
            notes = alert.get("annotations", {})
            print(f"  [{alert.get('status').upper()}] {labels.get('alertname')} "
                  f"instance={labels.get('instance')} severity={labels.get('severity')}")
            print(f"      {notes.get('summary', '')}")
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"ok")

    def log_message(self, *args):      # silence default access log
        pass


if __name__ == "__main__":
    print("Webhook receiver listening on :5001 ...")
    HTTPServer(("0.0.0.0", 5001), Handler).serve_forever()
```

Run: `python3 webhook_receiver.py` and use receiver:

```yaml
receivers:
  - name: lab-webhook
    webhook_configs:
      - url: 'http://localhost:5001/'
        send_resolved: true
```

When an alert fires you'll see lines like `[FIRING] HighCPUUsage instance=localhost:9100 severity=warning` printed, which is your **evidence of alert generation**.

---

# 1️⃣1️⃣ SLO-BASED ALERTING: BURN-RATE ALERTS ⭐⭐⭐ (ADVANCED, HIGH VALUE)

Threshold alerts (CPU > 80%) are **cause-based**. SLO alerts are **symptom-based** and tied to user impact (Part 1 error budgets).

## 11.1 The idea

```
SLO 99.9% over 30 days → error budget = 0.1% of requests
BURN RATE = (current error ratio) ÷ (allowed error ratio = 1 − SLO)
  burn rate 1   → you'll use exactly the whole budget in 30 days (fine)
  burn rate 14.4 → you'll exhaust the budget in 30d / 14.4 ≈ 2 days; in 1 hour you'd spend 2% of the monthly budget
```

## 11.2 Multi-window, multi-burn-rate alerts (Google SRE Workbook pattern)

| Severity | Burn rate | Long window | Short window | Budget consumed before alert | Action |
|---|---|---|---|---|---|
| **Page** (fast burn) | 14.4 | 1 h | 5 m | 2% | Wake someone up |
| **Page** (medium burn) | 6 | 6 h | 30 m | 5% | Wake someone up |
| **Ticket** (slow burn) | 1 | 3 d | 6 h | 10% | Create a ticket, work in hours |

Two windows: the **long** window avoids false alarms on blips; the **short** window makes the alert **reset quickly** once the problem is fixed.

## 11.3 Rules (SLO 99.9% for the Flask app)

Recording rules (error ratio at several windows), `rules/slo.yml`:

```yaml
groups:
  - name: slo_recording
    interval: 30s
    rules:
      - record: job:myapp_error_ratio:rate5m
        expr: sum by (job) (rate(myapp_http_requests_total{status=~"5.."}[5m])) / sum by (job) (rate(myapp_http_requests_total[5m]))
      - record: job:myapp_error_ratio:rate30m
        expr: sum by (job) (rate(myapp_http_requests_total{status=~"5.."}[30m])) / sum by (job) (rate(myapp_http_requests_total[30m]))
      - record: job:myapp_error_ratio:rate1h
        expr: sum by (job) (rate(myapp_http_requests_total{status=~"5.."}[1h])) / sum by (job) (rate(myapp_http_requests_total[1h]))
      - record: job:myapp_error_ratio:rate6h
        expr: sum by (job) (rate(myapp_http_requests_total{status=~"5.."}[6h])) / sum by (job) (rate(myapp_http_requests_total[6h]))
      - record: job:myapp_error_ratio:rate3d
        expr: sum by (job) (rate(myapp_http_requests_total{status=~"5.."}[3d])) / sum by (job) (rate(myapp_http_requests_total[3d]))

  - name: slo_alerts
    rules:
      - alert: ErrorBudgetBurnFast
        expr: job:myapp_error_ratio:rate1h > (14.4 * 0.001) and job:myapp_error_ratio:rate5m > (14.4 * 0.001)
        labels: { severity: critical }
        annotations:
          summary: "{{ $labels.job }} is burning the 99.9% error budget at >14.4x (2% of budget per hour)"

      - alert: ErrorBudgetBurnMedium
        expr: job:myapp_error_ratio:rate6h > (6 * 0.001) and job:myapp_error_ratio:rate30m > (6 * 0.001)
        labels: { severity: critical }
        annotations:
          summary: "{{ $labels.job }} is burning the error budget at >6x (5% of budget in 6 hours)"

      - alert: ErrorBudgetBurnSlow
        expr: job:myapp_error_ratio:rate3d > (1 * 0.001) and job:myapp_error_ratio:rate6h > (1 * 0.001)
        labels: { severity: warning }
        annotations:
          summary: "{{ $labels.job }} is steadily consuming its error budget (burn rate >1)"
```

(Tools such as **Sloth** and **Pyrra** generate these rules from a short SLO definition.)

## 11.4 Why this beats simple thresholds

```
✔ Alerts reflect USER impact and BUDGET, not arbitrary numbers
✔ Fast burn pages immediately; slow burn waits as a ticket
✔ Fewer false positives (two windows) and quick resets
✔ Gives teams a shared reliability language (Unit III DORA/SRE links)
```

---

# 1️⃣2️⃣ ALERT DESIGN BEST PRACTICES ✅

```
1. Alert on SYMPTOMS users feel (errors, latency, SLO burn); use dashboards for causes. Page only for urgent, actionable problems.
2. Every alert needs: clear summary, severity, owner/team label, runbook_url, dashboard link.
3. Use `for:` to filter transient spikes; use `keep_firing_for` to reduce flapping.
4. Prefer ratios/percentages to absolute numbers; use predict_linear for "will fill soon" (proactive).
5. Layer severities: critical (page), warning (ticket/Slack), info (dashboard).
6. Group, inhibit and route so one incident = one notification stream, to the right team.
7. Include the pipeline health: Watchdog heartbeat + alerts on `up`, rule-evaluation failures, notification failures.
8. Review alerts regularly: delete noisy, never-actionable alerts; tune thresholds from real data.
9. Test rules (promtool unit tests) and treat alert rules as code: Git, PRs, CI (Unit III).
10. Keep secrets out of configs; protect Alertmanager (it can be used to silence everything!).
11. Track on-call health: pages per shift, time to acknowledge, false-positive rate.
```

## 12.1 Anti-patterns

| Anti-pattern | Problem | Better |
|---|---|---|
| Paging on CPU > 80% | CPU spikes are normal; no user impact | Alert on latency/errors; warning-level ticket for sustained saturation |
| No `for:` | Flapping | `for: 5m` |
| Alerting without runbooks | Slow, inconsistent response | Add `runbook_url` |
| One giant receiver for everything | Noise for everyone | Route by team/severity |
| Alerts on every metric "just in case" | Alert fatigue | Few, meaningful alerts |
| Silences with no expiry/ownership | Forgotten muted problems | Expiry + comment + author |
| No Watchdog/meta-alerts | Broken alerting goes unnoticed | Heartbeat + external check |
| Hard-coded thresholds copy-pasted | Wrong for different services | Tune per service/SLO |

---

# 1️⃣3️⃣ TROUBLESHOOTING TABLE 🛠️

| Symptom | Likely cause | Fix |
|---|---|---|
| Rule file ignored | Not listed in `rule_files` / path wrong | Check glob; `promtool check config`; reload Prometheus |
| Alert stays **pending** | `for:` not yet satisfied or expr flickers | Wait; inspect expr in the Graph view; adjust `for` |
| Alert **never appears** | Expression returns nothing (label mismatch, wrong metric) | Run `expr` in the Query tab; fix selectors |
| Alert **firing** but no notification | Prometheus not configured with Alertmanager / AM unreachable | `alerting:` block; **Status → Runtime** shows AM; check `prometheus_notifications_dropped_total`, AM logs |
| Alertmanager received it but nothing sent | Route doesn't match → default receiver; silenced/inhibited; `group_wait` not elapsed | `amtool config routes test`; check Silences; wait `group_wait` |
| Email not delivered | SMTP auth/TLS/app password, port blocked | Check AM logs, use app password, port 587 egress; test with the webhook receiver first |
| Slack not delivered | Wrong webhook URL/channel; egress blocked | Test the URL with `curl -X POST -d '{"text":"test"}'` |
| Duplicate notifications | HA Prometheus without AM clustering or AM not peered | Cluster Alertmanagers; send to all instances |
| Notification storm | `group_by` too fine; no inhibition | `group_by: [alertname, cluster]`; add inhibit rules |
| Reminder emails every 4 hours | Default `repeat_interval` | Increase it or fix the root cause |
| `yaml: line N: mapping values are not allowed` | Indentation/colon issues | `amtool check-config`; use spaces |
| Resolved messages missing | `send_resolved: false` (default for some receivers) | Set `send_resolved: true` |
| Template error `executing "…" at <.Foo>` | Wrong template field | Check data structure docs; test with simple text |

---

# 1️⃣4️⃣ ALERTING IN THE WIDER WORKFLOW 🔗

```
Alert fires → Alertmanager routes to on-call (PagerDuty/Slack) → engineer opens dashboard (Grafana, Part 5) from the alert link
  → drills into logs/traces (Part 7) → mitigates (rollback via Jenkins/feature flag, Unit III) → resolves → blameless postmortem
  → improves: new alert/runbook/test; error budget updated (SLO governance, Unit VI)
```

Alert-driven automation examples: scale-out via webhook to Lambda; auto-restart; ChatOps commands; ticket creation (Jira) from alerts.

---

# 1️⃣5️⃣ VIVA QUESTIONS (PART 4) 🎯

**Q1. How does alerting work in Prometheus?** Prometheus evaluates alerting rules; firing alerts are sent to Alertmanager, which groups, routes, silences and sends notifications to receivers.

**Q2. What is the role of `for` in an alert rule?** The expression must be true continuously for that duration before the alert fires; prevents flapping.

**Q3. Explain alert states.** Inactive (false), pending (true but within `for`), firing (true for ≥ `for`), then resolved.

**Q4. What are labels vs annotations in a rule?** Labels (severity, team) identify and route the alert; annotations hold descriptive text/links with templates.

**Q5. What does Alertmanager do?** Deduplicates, groups, routes, inhibits, silences and delivers notifications; supports HA clustering.

**Q6. Explain `group_by`, `group_wait`, `group_interval`, `repeat_interval`.** Grouping labels; initial wait for a new group; wait before sending updates for an existing group; reminder frequency while firing.

**Q7. Inhibition vs silence?** Inhibition automatically mutes target alerts while a source alert fires; a silence is a manual, time-limited mute.

**Q8. How do you route critical alerts to PagerDuty and warnings to Slack?** Child routes with matchers `severity="critical"` → pager receiver and `severity="warning"` → Slack receiver.

**Q9. How can you test alert rules?** `promtool check rules` for syntax and `promtool test rules` with synthetic series, plus triggering real conditions (e.g., CPU stress).

**Q10. What is alert fatigue and how do you reduce it?** Ignoring alerts due to noise; use symptom-based alerts, `for`, grouping, inhibition, routing and regular review.

**Q11. What is a Watchdog alert?** An always-firing alert that proves the alerting pipeline works; if it stops arriving, alerting is broken.

**Q12. What is burn-rate alerting?** Alerting on how fast the SLO error budget is being consumed, using multiple windows (e.g., 14.4× over 1 h and 5 min) to page fast and avoid false positives.

**Q13. Why send Prometheus alerts to all Alertmanager instances?** Alertmanager clustering deduplicates; each Prometheus must reach every instance for resilience.

**Q14. What is `amtool` used for?** Managing alerts and silences, validating configs and testing routing from the command line.

**Q15. Where should secrets like SMTP passwords be stored?** In protected files (`*_file` options) or secret managers, not plaintext in the config repository.

---

# 🧠 PART 4 MASTER SHEET

```
ALERTING = Prometheus (rules: WHEN) + Alertmanager (WHO/HOW) ; AM :9093 (cluster :9094)
RULE: alert, expr, for, keep_firing_for, labels (route), annotations (summary, description, runbook_url) ; templates $labels, $value, humanize*
STATES: inactive → pending (waiting `for`) → firing → resolved ; ALERTS metric ; Prometheus re-sends every minute
RULE LIBRARY: Watchdog, InstanceDown (up==0), HighCPU, HighMemory, DiskSpaceLow, DiskWillFill (predict_linear), HighLoad, HighErrorRate, HighLatencyP95
TEST: promtool check rules | promtool test rules (input_series, alert_rule_test) | reload via /-/reload
ALERTMANAGER: dedupe • group (group_by) • route (matchers tree, continue) • inhibit (source/target/equal) • silence • notify (send_resolved) • HA gossip
TIMING: group_wait 30s | group_interval 5m | repeat_interval 4h
RECEIVERS: email, Slack, PagerDuty, webhook, SNS, Telegram... ; secrets in *_file ; time_intervals for office hours/maintenance
amtool: alert, silence add/query/expire, check-config, config routes test
SLO ALERTS: burn rate = error ratio ÷ (1−SLO) ; multi-window: 14.4× (1h&5m) page, 6× (6h&30m) page, 1× (3d&6h) ticket
BEST PRACTICE: symptom-based, actionable, runbooks, `for`, group/inhibit/route, Watchdog, review & prune, rules as code, protect Alertmanager
```

---

## 🫡 PART 4 STATUS

You can write, test and load alert rules, configure Alertmanager routing, grouping, inhibition, silences and receivers, run a webhook receiver for lab proof, and design SLO burn-rate alerts.

**STOP here.** Next up:

> **PART 5 — Grafana and Performance Dashboards (data sources, panels, variables, dashboard design, provisioning as code, annotations, alerting) 📊**
