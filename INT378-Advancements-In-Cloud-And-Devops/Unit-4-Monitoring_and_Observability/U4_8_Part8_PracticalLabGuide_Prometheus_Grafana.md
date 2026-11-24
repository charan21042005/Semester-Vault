# 🧪 UNIT IV — PART 8: PRACTICAL LAB GUIDE (PROMETHEUS + NODE EXPORTER + ALERTS, THEN GRAFANA DASHBOARDS)

> **Covered in this document:** lab goal, architecture, cost and safety rules, and what was **tested** → **Step 1** launch the EC2 server and security group → **Step 2** prepare the OS and service users → **Practical 8 (Prometheus)**: **Step 3** install Node Exporter (systemd) → **Step 4** install Prometheus (systemd) → **Step 5** configure scraping, validate with `promtool`, open the UI → **Step 6** explore CPU, memory, disk and network metrics with PromQL → **Step 7** install Alertmanager and a webhook receiver → **Step 8** define and unit-test alert rules → **Step 9** generate real load and **validate alert generation** (CPU, memory, disk, instance down, inhibition, silences) → optional extras (second host, Flask app target, e-mail receiver) → **Practical 9 (Grafana)**: **Step 10** install Grafana → **Step 11** add Prometheus as a data source → **Step 12** import a ready-made dashboard and the supplied lab dashboard → **Step 13** design your own infrastructure dashboard panel by panel → **Step 14** generate load and **analyze performance trends** → optional Grafana alert → troubleshooting tables → evidence checklists → cleanup → report templates → practical viva → quick checklists.

---

Absolutely mate. 😤🔥 This is the lab manual for **Practical 8 and Practical 9**. Your syllabus lines:

> **P8:** *Install and configure Prometheus, deploy Node Exporter, collect CPU, memory, disk, and network metrics, define alerting rules, and validate alert generation for infrastructure monitoring.*
>
> **P9:** *Deploy Grafana, integrate it with Prometheus as a data source, design monitoring dashboards for infrastructure performance, visualize CPU, memory, storage, and network utilization, and analyze system performance trends.*

---

# 0️⃣ GOAL, ARCHITECTURE, RULES AND WHAT WAS TESTED

## 0.1 What you will build (one server)

```
                       ┌───────────────────────── EC2 Ubuntu 24.04 (t3.small) ─────────────────────────┐
 Your browser ──:3000──▶│ GRAFANA ──PromQL──▶ PROMETHEUS :9090 ──scrapes every 15s──▶ NODE EXPORTER :9100 │
 Your browser ──:9090──▶│                         │  evaluates alert rules                                │
 Your browser ──:9093──▶│                         ▼ firing alerts                                         │
                       │                    ALERTMANAGER :9093 ──webhook──▶ lab webhook receiver :5001   │
                       └─────────────────────────────────────────────────────────────────────────────────┘
 (Optional: a 2nd EC2 instance running Node Exporter; the instrumented Flask app on :5000; e-mail receiver)
```

| Syllabus requirement | How we satisfy it |
|---|---|
| Install and configure Prometheus | Binary install + `prometheus.yml` + systemd service |
| Deploy Node Exporter | Binary install + systemd service on port 9100 |
| Collect CPU, memory, disk, network metrics | Scrape job `node`; PromQL queries in Step 6 |
| Define alerting rules and validate alert generation | `alerts.yml`, `promtool test`, real stress tests, webhook proof |
| Deploy Grafana + Prometheus data source | Grafana via apt; data source `http://localhost:9090` |
| Dashboards for CPU, memory, storage, network | Custom dashboard (Step 13) + supplied JSON + imported community dashboard |
| Analyze performance trends | Controlled load tests + trend analysis table (Step 14) |

## 0.2 Cost and safety rules 🚨

```
• Use t3.small (2 GB RAM) or larger. Terminate or stop the instance when finished. Public IPv4 + running EC2 cost money.
• Free Tier terms for new AWS accounts changed in 2025 (credit-based plan): check your Billing console.
• t3 instances are "burstable". Running a 100% CPU stress test for 5 minutes is fine, but don't leave stress running.
• Open ports 9090, 9093, 3000, 5000 ONLY to YOUR IP (Prometheus/Alertmanager have no authentication by default!).
• Disk and memory tests: always clean up (rm the big file, stop stress-ng).
• Never expose 9100/9090/9093 to 0.0.0.0/0.
```

## 0.3 What was tested (honesty note) ✅

The lab files were validated before being given to you: **`promtool check config/rules`**, **`promtool test rules`** and **`amtool check-config`** all pass; the full stack (Node Exporter, Prometheus 3.13, Alertmanager 0.32, the webhook receiver) was run end-to-end and a real alert went **pending → firing → delivered to the webhook**; every PromQL query in this lab and every panel query in the supplied dashboard JSON returned data against live Node Exporter metrics; and the instrumented Flask app exported its metrics correctly. **Not tested here:** the systemd unit behavior on a real EC2 instance and the Grafana UI import (Grafana isn't available in my test environment). Follow the steps and use the troubleshooting tables if something differs on your machine.

## 0.4 Versions

The steps use variables so you can change versions without editing commands. Check the current releases:

```
Prometheus:   https://prometheus.io/download/   (use the current LTS or latest; tested: 3.13.0)
Alertmanager: https://prometheus.io/download/   (tested: 0.32.0)
Node Exporter:https://prometheus.io/download/   (tested: 1.11.1)
Grafana:      https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/
```

## 0.5 Lab files bundle

You also received **`U4_lab_files.zip`** containing every config used below (`prometheus.yml`, `rules/alerts.yml`, `rules/recording.yml`, `tests/alerts_test.yml`, `alertmanager.yml`, `webhook_receiver.py`, `app_instrumented.py`, `systemd/*.service`, `grafana/infra-dashboard.json`) and **`U4_infra-dashboard.json`** (the dashboard). You can copy them to the server with `scp -i lab-key.pem -r U4_lab_files ubuntu@<IP>:~/` or paste the file contents as shown in each step.

---

# 1️⃣ STEP 1: LAUNCH THE EC2 SERVER AND SECURITY GROUP

| Field | Value |
|---|---|
| Name | `monitoring-server` |
| AMI | **Ubuntu Server 24.04 LTS** (x86_64; for Graviton `t4g` use the `arm64` downloads) |
| Instance type | **t3.small** (t3.medium if you also run Docker/Jenkins) |
| Key pair | `lab-key` (Unit I Part 8) |
| Network | Default VPC, public subnet, auto-assign public IP **enabled** |
| Storage | **20 GiB** gp3 (needed for the disk-usage test) |
| Security group | `monitoring-sg` |

**Inbound rules (all from "My IP" unless noted):**

| Port | Purpose |
|---|---|
| 22 | SSH |
| 9090 | Prometheus UI |
| 9093 | Alertmanager UI |
| 3000 | Grafana |
| 5000 | Flask demo app (optional) |
| 9100 | Node Exporter: only needed if a **second** server must be scraped → allow from the Prometheus server's **security group** (not from the internet) |

Connect:

```bash
chmod 400 lab-key.pem
ssh -i lab-key.pem ubuntu@<EC2_PUBLIC_IP>
```

📸 Screenshot: instance running + security group rules.

---

# 2️⃣ STEP 2: PREPARE THE OS

```bash
sudo apt update && sudo apt -y upgrade
sudo apt install -y wget curl jq python3 python3-pip stress-ng
uname -m                     # x86_64 → use linux-amd64 downloads ; aarch64 → use linux-arm64
free -h && df -h /           # note baseline memory and disk (you will compare later)
```

Create dedicated **non-login service users** (security best practice: services don't run as root):

```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin node_exporter
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus
sudo useradd --no-create-home --shell /usr/sbin/nologin alertmanager
```

---

# 🅰️ PRACTICAL 8: PROMETHEUS, NODE EXPORTER AND ALERTING

# 3️⃣ STEP 3: INSTALL NODE EXPORTER

Node Exporter exposes the host's CPU, memory, disk and network metrics at `:9100/metrics`.

```bash
NE_VER=1.11.1                                  # check prometheus.io/download for the current version
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v${NE_VER}/node_exporter-${NE_VER}.linux-amd64.tar.gz
wget -q https://github.com/prometheus/node_exporter/releases/download/v${NE_VER}/sha256sums.txt
sha256sum -c sha256sums.txt --ignore-missing   # should print: node_exporter-...tar.gz: OK
tar xzf node_exporter-${NE_VER}.linux-amd64.tar.gz
sudo cp node_exporter-${NE_VER}.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
node_exporter --version
```

Create the systemd service:

```bash
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<'EOF'
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter --no-pager       # active (running)
```

**Verify:**

```bash
curl -s localhost:9100/metrics | head -20
curl -s localhost:9100/metrics | grep -E "^(node_cpu_seconds_total|node_memory_MemAvailable_bytes|node_filesystem_avail_bytes|node_network_receive_bytes_total|node_load1) " | head -10
curl -s localhost:9100/metrics | wc -l               # roughly 1000+ lines
```

📸 Screenshot: `systemctl status node_exporter` and the metrics output. In your report, explain the **types**: `node_cpu_seconds_total` is a **counter**, `node_memory_MemAvailable_bytes` is a **gauge** (Part 2).

---

# 4️⃣ STEP 4: INSTALL PROMETHEUS

```bash
PROM_VER=3.13.0                                 # LTS line tested; check prometheus.io/download
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VER}/prometheus-${PROM_VER}.linux-amd64.tar.gz
wget -q https://github.com/prometheus/prometheus/releases/download/v${PROM_VER}/sha256sums.txt
sha256sum -c sha256sums.txt --ignore-missing
tar xzf prometheus-${PROM_VER}.linux-amd64.tar.gz

sudo mkdir -p /etc/prometheus/rules /var/lib/prometheus
sudo cp prometheus-${PROM_VER}.linux-amd64/prometheus prometheus-${PROM_VER}.linux-amd64/promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
prometheus --version
```

(The Prometheus 3.x tarball contains `prometheus`, `promtool`, a sample `prometheus.yml`, `LICENSE` and `NOTICE`; older console template directories are no longer bundled.)

Create the systemd service:

```bash
sudo tee /etc/systemd/system/prometheus.service > /dev/null <<'EOF'
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=15d \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

Flags explained: `--config.file` (config), `--storage.tsdb.path` (data directory), `--storage.tsdb.retention.time=15d` (how long to keep data), `--web.listen-address` (bind address; restrict with the Security Group), `--web.enable-lifecycle` (allows `POST /-/reload`).

---

# 5️⃣ STEP 5: CONFIGURE SCRAPING, VALIDATE, START

## 5.1 `/etc/prometheus/prometheus.yml`

```bash
sudo tee /etc/prometheus/prometheus.yml > /dev/null <<'EOF'
# /etc/prometheus/prometheus.yml  (Practical 8)
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: "lab-prometheus"
    env: "dev"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["localhost:9093"]

rule_files:
  - "/etc/prometheus/rules/*.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["localhost:9100"]
        labels:
          role: "monitoring-server"
      # add more hosts later, e.g. "10.0.1.20:9100"

  - job_name: "alertmanager"
    static_configs:
      - targets: ["localhost:9093"]

  # optional (Part 2 instrumented Flask app)
  # - job_name: "flask-app"
  #   static_configs:
  #     - targets: ["localhost:5000"]
EOF
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

Explain each block in your report: `global` (15 s scrape and evaluation; `external_labels` identify this Prometheus), `alerting` (where Alertmanager is), `rule_files`, and three `scrape_configs` jobs (Prometheus monitors itself; the `node` job scrapes Node Exporter; Alertmanager's own metrics).

## 5.2 Validate and start

```bash
promtool check config /etc/prometheus/prometheus.yml
# → SUCCESS: prometheus.yml is valid prometheus config file syntax  (a warning about rule files not existing yet is fine; we add them in Step 8)

sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus --no-pager
curl -s localhost:9090/-/healthy      # Prometheus Server is Healthy.
curl -s localhost:9090/-/ready        # Prometheus Server is Ready.
sudo journalctl -u prometheus -n 20 --no-pager      # check for errors
```

## 5.3 Open the UI and check targets ⭐

Browser: **`http://<EC2_PUBLIC_IP>:9090`**

```
Status → Targets    →  jobs "prometheus", "node" (UP, green) ; "alertmanager" will be DOWN until Step 7
Query tab           →  run:  up      (should show 1 for prometheus and node)
```

📸 Screenshot: **Status → Targets** and the `up` query result.

---

# 6️⃣ STEP 6: COLLECT AND EXPLORE CPU, MEMORY, DISK AND NETWORK METRICS

In the Prometheus **Query** tab (use **Table** for instant values and **Graph** for trends). All queries below were verified against live Node Exporter data.

| Resource | Query | Meaning / expected |
|---|---|---|
| **CPU usage %** | `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)` | Near 0-5% when idle |
| **CPU by mode** | `avg by (mode) (rate(node_cpu_seconds_total{mode!="idle"}[1m])) * 100` | user, system, iowait, steal... |
| **Number of cores** | `count by (instance) (node_cpu_seconds_total{mode="idle"})` | 2 on a t3.small |
| **Load per core** | `node_load1 / on(instance) count by (instance) (node_cpu_seconds_total{mode="idle"})` | Sustained ≥ 1 = saturation |
| **Memory used %** | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100` | Typically 15-30% idle |
| **Memory available (MiB)** | `node_memory_MemAvailable_bytes / 1024 / 1024` | Honest "free" memory |
| **Disk used % (root)** | `100 - (node_filesystem_avail_bytes{mountpoint="/", fstype!~"tmpfs\|overlay"} / node_filesystem_size_bytes{mountpoint="/", fstype!~"tmpfs\|overlay"} * 100)` | Low on a fresh instance (about 10-20%) |
| **Disk free (GiB)** | `node_filesystem_avail_bytes{mountpoint="/", fstype!~"tmpfs\|overlay"} / 1024 / 1024 / 1024` | |
| **Disk I/O** | `rate(node_disk_written_bytes_total[1m])`, `rate(node_disk_read_bytes_total[1m])`, `rate(node_disk_io_time_seconds_total[1m]) * 100` | Throughput and utilization |
| **Network in/out (bits/s)** | `sum by (instance) (rate(node_network_receive_bytes_total{device!="lo"}[1m])) * 8` and `..._transmit_bytes_total...` | Small when idle |
| **Uptime (s)** | `time() - node_boot_time_seconds` | Seconds since boot |
| **Targets down** | `count(up == 0) or vector(0)` | 0 when all is well |

(In the Prometheus UI, type the `\|` characters as a plain `|`; the backslashes above only prevent the Markdown table from breaking.)

Take **baseline screenshots** of CPU %, memory %, disk %, network graphs. You will compare them after the load tests.

Optional **recording rules** (precomputed queries for dashboards): copy `rules/recording.yml` to `/etc/prometheus/rules/` (from the bundle), `promtool check rules`, reload Prometheus; then `instance:node_cpu_utilisation:rate5m` becomes a queryable series.

---

# 7️⃣ STEP 7: INSTALL ALERTMANAGER AND A WEBHOOK RECEIVER

## 7.1 Alertmanager

```bash
AM_VER=0.32.0                                 # check prometheus.io/download
cd /tmp
wget https://github.com/prometheus/alertmanager/releases/download/v${AM_VER}/alertmanager-${AM_VER}.linux-amd64.tar.gz
wget -q https://github.com/prometheus/alertmanager/releases/download/v${AM_VER}/sha256sums.txt
sha256sum -c sha256sums.txt --ignore-missing
tar xzf alertmanager-${AM_VER}.linux-amd64.tar.gz

sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
sudo cp alertmanager-${AM_VER}.linux-amd64/alertmanager alertmanager-${AM_VER}.linux-amd64/amtool /usr/local/bin/
sudo chown alertmanager:alertmanager /usr/local/bin/alertmanager /usr/local/bin/amtool
sudo chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager
alertmanager --version
```

## 7.2 `/etc/alertmanager/alertmanager.yml` (webhook receiver; no e-mail setup needed)

```bash
sudo tee /etc/alertmanager/alertmanager.yml > /dev/null <<'EOF'
# /etc/alertmanager/alertmanager.yml  (Practical 8: webhook receiver, no email needed)
global:
  resolve_timeout: 5m

route:
  receiver: lab-webhook
  group_by: ["alertname", "instance"]
  group_wait: 10s              # production: 30s
  group_interval: 1m           # production: 5m
  repeat_interval: 1h          # production: 4h
  routes:
    - matchers:
        - alertname="Watchdog"
      receiver: "null"

inhibit_rules:
  - source_matchers:
      - alertname="InstanceDown"
    target_matchers:
      - alertname=~"HighCPUUsage|HighMemoryUsage|DiskUsageHigh|HighLoadPerCore"
    equal: ["instance"]

receivers:
  - name: "null"
  - name: lab-webhook
    webhook_configs:
      - url: "http://localhost:5001/"
        send_resolved: true
EOF
sudo chown alertmanager:alertmanager /etc/alertmanager/alertmanager.yml
amtool check-config /etc/alertmanager/alertmanager.yml          # SUCCESS, 1 inhibit rule, 2 receivers
amtool config routes test --config.file=/etc/alertmanager/alertmanager.yml alertname=HighCPUUsage instance=x    # → lab-webhook
amtool config routes test --config.file=/etc/alertmanager/alertmanager.yml alertname=Watchdog                    # → null
```

## 7.3 Alertmanager service

```bash
sudo tee /etc/systemd/system/alertmanager.service > /dev/null <<'EOF'
[Unit]
Description=Prometheus Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --cluster.listen-address=
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

(`--cluster.listen-address=` with an empty value **disables HA gossip** for this single-node lab. Without it, Alertmanager may refuse to start on hosts that have no private IP address. On a normal EC2 instance it would also work, but disabling clustering keeps the lab simple.)

## 7.4 The lab webhook receiver (proof of alert delivery)

```bash
sudo mkdir -p /opt/lab
sudo tee /opt/lab/webhook_receiver.py > /dev/null <<'EOF'
"""Tiny Alertmanager webhook receiver for the lab: prints alerts it receives."""
import json
from datetime import datetime
from http.server import BaseHTTPRequestHandler, HTTPServer


class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        length = int(self.headers.get("Content-Length", 0))
        payload = json.loads(self.rfile.read(length) or b"{}")
        print("\n" + "=" * 70, flush=True)
        print(datetime.now().isoformat(timespec="seconds"),
              "| status:", payload.get("status"),
              "| receiver:", payload.get("receiver"), flush=True)
        for alert in payload.get("alerts", []):
            labels = alert.get("labels", {})
            notes = alert.get("annotations", {})
            print(f"  [{alert.get('status', '').upper()}] {labels.get('alertname')} "
                  f"instance={labels.get('instance')} severity={labels.get('severity')}", flush=True)
            print(f"      {notes.get('summary', '')}", flush=True)
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"ok")

    def log_message(self, *args):
        pass


if __name__ == "__main__":
    print("Webhook receiver listening on :5001 ...", flush=True)
    HTTPServer(("0.0.0.0", 5001), Handler).serve_forever()
EOF

sudo tee /etc/systemd/system/lab-webhook.service > /dev/null <<'EOF'
[Unit]
Description=Lab webhook receiver for Alertmanager
After=network-online.target

[Service]
User=nobody
Type=simple
ExecStart=/usr/bin/python3 /opt/lab/webhook_receiver.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now alertmanager lab-webhook
systemctl is-active alertmanager lab-webhook          # active active
curl -s localhost:9093/-/healthy                       # OK
```

Keep a **second SSH terminal** open watching the receiver:

```bash
sudo journalctl -u lab-webhook -f
```

Verify Prometheus knows Alertmanager:

```bash
curl -s localhost:9090/api/v1/alertmanagers | jq        # activeAlertmanagers → http://localhost:9093/api/v2/alerts
```

Alertmanager UI: **`http://<EC2_PUBLIC_IP>:9093`**. In Prometheus **Status → Targets** the `alertmanager` job is now UP.

---

# 8️⃣ STEP 8: DEFINE AND UNIT-TEST ALERT RULES ⭐

## 8.1 `/etc/prometheus/rules/alerts.yml` (lab thresholds are intentionally low)

```bash
sudo tee /etc/prometheus/rules/alerts.yml > /dev/null <<'EOF'
# /etc/prometheus/rules/alerts.yml
# LAB thresholds are deliberately low so alerts fire within minutes. See the comments for production values.
groups:
  - name: availability
    rules:
      - alert: Watchdog
        expr: vector(1)
        labels:
          severity: none
        annotations:
          summary: "Watchdog: alerting pipeline is alive"

      - alert: InstanceDown
        expr: up == 0
        for: 30s                      # production: 2m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.job }}/{{ $labels.instance }} has been unreachable for more than 30 seconds."

  - name: node_resources
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100) > 50     # production: > 80 for 5m
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | printf \"%.1f\" }}% (lab threshold 50%)."

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 70               # production: > 85 for 5m
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | printf \"%.1f\" }}% (lab threshold 70%)."

      - alert: DiskUsageHigh
        expr: |
          100 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs"}
                 / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs"} * 100) > 40       # production: > 85 for 10m
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Disk usage high on {{ $labels.instance }} ({{ $labels.mountpoint }})"
          description: "{{ $labels.mountpoint }} is {{ $value | printf \"%.1f\" }}% full (lab threshold 40%)."

      - alert: DiskWillFillIn4Hours
        expr: predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs|ramfs"}[1h], 4 * 3600) < 0
        for: 30m
        labels:
          severity: critical
        annotations:
          summary: "Disk on {{ $labels.instance }} ({{ $labels.mountpoint }}) predicted to fill within 4 hours"

      - alert: HighLoadPerCore
        expr: node_load5 / on(instance) count by (instance) (node_cpu_seconds_total{mode="idle"}) > 1.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High load on {{ $labels.instance }}"
          description: "5-minute load per core is {{ $value | printf \"%.2f\" }}."
EOF
sudo chown prometheus:prometheus /etc/prometheus/rules/alerts.yml
```

Explain in the report: `expr` (PromQL condition), `for` (pending time before firing), `labels` (`severity` drives routing), `annotations` (templated human text), and why `Watchdog` always fires (pipeline heartbeat).

> ⚠️ **DiskUsageHigh note:** on a 20 GiB root disk, the lab threshold of 40% is crossed only after you write a big file (Step 9). If your fresh instance already shows > 40% used, raise the threshold (e.g., to 60) in this lab rule.

## 8.2 Validate and load

```bash
promtool check rules /etc/prometheus/rules/alerts.yml           # SUCCESS: 7 rules found
promtool check config /etc/prometheus/prometheus.yml             # SUCCESS: rule files found
curl -X POST localhost:9090/-/reload                             # (or: sudo systemctl reload prometheus)
```

Open **Prometheus → Alerts**: rules appear green (inactive), `Watchdog` is **firing**; **Status → Rules** shows each rule group with evaluation time.

## 8.3 Unit-test the rule with `promtool` (professional touch)

```bash
mkdir -p ~/prom-tests && cd ~/prom-tests
cat > alerts_test.yml <<'EOF'
rule_files:
  - /etc/prometheus/rules/alerts.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'up{job="node", instance="a:9100"}'
        values: '1 1 0 0 0 0 0 0'
    alert_rule_test:
      - eval_time: 1m
        alertname: InstanceDown
        exp_alerts: []
      - eval_time: 5m
        alertname: InstanceDown
        exp_alerts:
          - exp_labels:
              severity: critical
              job: node
              instance: "a:9100"
            exp_annotations:
              summary: "Instance a:9100 is down"
              description: "node/a:9100 has been unreachable for more than 30 seconds."
EOF
promtool test rules alerts_test.yml          # Unit Testing: alerts_test.yml   SUCCESS
```

This proves the rule logic with **synthetic data** (down for 3+ minutes → firing) before touching production.

📸 Screenshot: Alerts page (Watchdog firing, others inactive) and the `promtool test rules` SUCCESS.

---

# 9️⃣ STEP 9: GENERATE LOAD AND VALIDATE ALERT GENERATION ⭐⭐⭐

You will trigger **five scenarios** and observe the alert at **four places**:

```
WHERE TO OBSERVE (for every scenario)
 1. Prometheus → Alerts page          (inactive → PENDING → FIRING)
 2. Prometheus query:  ALERTS{alertname="HighCPUUsage"}                      (alertstate="pending"/"firing")
 3. Alertmanager UI :9093              (the alert appears, grouped, with labels/annotations)
 4. Webhook receiver (journalctl -u lab-webhook -f)   ← the NOTIFICATION actually delivered
```

Timeline to expect (CPU example): stress starts → within ~30-60 s the 1-minute CPU average passes 50% → **pending** → after `for: 1m` → **firing** → Alertmanager waits `group_wait: 10s` → **webhook receives FIRING**. Total ≈ 2-3 minutes. When the load stops: alert clears → **RESOLVED** webhook message.

## 9.1 Scenario 1: High CPU 🔥

```bash
stress-ng --cpu 0 --timeout 300s --metrics-brief        # all cores busy for 5 minutes
```

Observe: in Prometheus graph `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)` climbs toward 100%. Expected webhook output:

```
======================================================================
2026-11-24T10:31:12 | status: firing | receiver: lab-webhook
  [FIRING] HighCPUUsage instance=localhost:9100 severity=warning
      High CPU usage on localhost:9100
```

After the test ends (5 min) you receive `[RESOLVED] HighCPUUsage`.

## 9.2 Scenario 2: High memory

```bash
free -h                                                  # note available memory first
stress-ng --vm 1 --vm-bytes 1G --vm-keep --timeout 300s   # t3.small has ~1.9 GiB total: 1 GiB pushes usage above 70%
```

(On larger instances raise `--vm-bytes` until usage passes 70%, e.g. 60-70% of total RAM. If the instance becomes sluggish, stop the test with Ctrl+C.) Expected: `HighMemoryUsage` firing; resolves after the timeout.

## 9.3 Scenario 3: Disk usage

```bash
df -h /                                                  # note baseline % used
sudo fallocate -l 7G /var/tmp/fill.img                    # adds 7 GiB (instant, sparse-free allocation)
df -h /                                                  # now above 40%
```

Expected: `DiskUsageHigh` firing for the mountpoint `/` after 1 minute. **Clean up immediately after the alert:**

```bash
sudo rm -f /var/tmp/fill.img
```

## 9.4 Scenario 4: Instance down (availability) + inhibition demo ⭐

```bash
sudo systemctl stop node_exporter
```

Expected within ~45-60 s: `up{job="node"} == 0` → **`InstanceDown`** (critical) FIRING in Prometheus, Alertmanager and the webhook.
**Inhibition demo:** while `InstanceDown` is firing, start a CPU stress in another terminal; `HighCPUUsage` would fire only if its metric existed, and the inhibit rule (source `InstanceDown`, targets include `HighCPUUsage`, `equal: [instance]`) suppresses resource alerts for that instance. (With Node Exporter stopped the CPU metric disappears anyway; the concept is demonstrated in your explanation.)

Restore:

```bash
sudo systemctl start node_exporter       # InstanceDown → RESOLVED notification after the target returns
```

## 9.5 Scenario 5: Silences and routing ⭐

1. Trigger CPU stress again for 3 minutes.
2. Before the notification, open **Alertmanager → Silences → New Silence**: matcher `alertname="HighCPUUsage"`, duration 30 minutes, comment "load test", author you.
3. Observe: alert is **active but silenced**; **no webhook message** arrives.
4. Expire the silence (or via CLI):

```bash
amtool --alertmanager.url=http://localhost:9093 silence add alertname=HighCPUUsage --duration=30m --comment="load test" --author="you"
amtool --alertmanager.url=http://localhost:9093 silence query
amtool --alertmanager.url=http://localhost:9093 silence expire <silence-id>
amtool --alertmanager.url=http://localhost:9093 alert
```

## 9.6 Evidence from the command line

```bash
curl -s localhost:9090/api/v1/alerts | jq '.data.alerts[] | {alertname: .labels.alertname, state, value}'
curl -s localhost:9093/api/v2/alerts | jq '.[] | {alert: .labels.alertname, status: .status.state, startsAt}'
```

## 9.7 Record your results ✍️

| Scenario | Trigger | Time to pending | Time to firing | Webhook received? | Resolved after |
|---|---|---|---|---|---|
| CPU | `stress-ng --cpu 0` | ___ s | ___ s | yes/no | ___ |
| Memory | `stress-ng --vm 1 --vm-bytes 1G` | | | | |
| Disk | `fallocate -l 7G` | | | | |
| Instance down | stop node_exporter | | | | |
| Silence | silenced alert | | | no (silenced) | |

📸 Screenshots: Prometheus Alerts page showing PENDING then FIRING; Alertmanager UI with the alert; terminal showing the webhook FIRING and RESOLVED lines; the graph spike for the corresponding metric.

---

# 🔟 OPTIONAL EXTRAS FOR PRACTICAL 8 (EXTRA MARKS) ⭐

## 10.1 Monitor a second server

```bash
# on the 2nd EC2 instance: install Node Exporter exactly as in Step 3
# security group of server 2: allow TCP 9100 FROM the monitoring server's security group
# on the monitoring server: add the target and reload
sudo sed -i 's#- targets: \["localhost:9100"\]#- targets: ["localhost:9100", "10.0.1.20:9100"]#' /etc/prometheus/prometheus.yml
promtool check config /etc/prometheus/prometheus.yml && curl -X POST localhost:9090/-/reload
```

(Use the second server's **private IP**; edit the file by hand if `sed` doesn't match your formatting.) Then compare hosts with `topk(2, ...)`, per-instance panels, and an `InstanceDown` test by stopping Node Exporter on server 2.

## 10.2 Monitor your own application (white-box)

Run the instrumented Flask app from Part 2 (`app_instrumented.py` in the bundle):

```bash
pip3 install --break-system-packages flask prometheus-client     # Ubuntu 24.04 uses PEP 668 "externally managed" Python
python3 app_instrumented.py &
for i in $(seq 1 60); do curl -s localhost:5000/order >/dev/null; curl -s localhost:5000/flaky >/dev/null; curl -s localhost:5000/slow >/dev/null; done
curl -s localhost:5000/metrics | grep myapp_ | head
```

Uncomment the `flask-app` job in `prometheus.yml`, reload, and query:

```promql
sum by (endpoint) (rate(myapp_http_requests_total[1m]))
100 * sum(rate(myapp_http_requests_total{status=~"5.."}[1m])) / sum(rate(myapp_http_requests_total[1m]))
histogram_quantile(0.95, sum by (le) (rate(myapp_http_request_duration_seconds_bucket[1m])))
```

(Note: `myapp_http_requests_in_progress` includes the in-flight `/metrics` scrape itself, so it reads at least 1 while Prometheus is scraping.) Add the `HighErrorRate` rule from Part 4 and trigger it by calling `/flaky` repeatedly (30% failures).

## 10.3 E-mail receiver

Create a Gmail **app password** (2-step verification on), save it to `/etc/alertmanager/smtp_password` (`chmod 600`, owner `alertmanager`), and replace the receiver with the commented e-mail block from the bundle's `alertmanager.yml` (`smarthost: smtp.gmail.com:587`). Validate with `amtool check-config`, restart Alertmanager, trigger an alert. (Outbound port 587 must be allowed; use the webhook receiver first to confirm the pipeline.)

## 10.4 Blackbox probe

Install `blackbox_exporter` and add the `blackbox-http` job from Part 2 to alert on `probe_success == 0` for your Jenkins/app URL.

---

# 1️⃣1️⃣ TROUBLESHOOTING (PRACTICAL 8) 🛠️

| Symptom | Likely cause | Fix |
|---|---|---|
| `sha256sum -c` says no file verified | Wrong file name/version | Check `ls` and version variable; the checksum file must be from the same release |
| `useradd: user already exists` | Re-run | Ignore |
| Service fails: `status=203/EXEC` | Binary missing/not executable/wrong path or CPU architecture | `ls -l /usr/local/bin`; `file /usr/local/bin/prometheus`; use `arm64` build on Graviton |
| Prometheus won't start: *permission denied* on `/var/lib/prometheus` | Wrong ownership | `sudo chown -R prometheus:prometheus /var/lib/prometheus /etc/prometheus` |
| Prometheus: *error loading config* / YAML error | Tabs/indent | `promtool check config`; use spaces |
| Target DOWN: *connection refused* | Exporter not running / wrong port | `systemctl status node_exporter`; `curl localhost:9100/metrics` |
| Target DOWN: *context deadline exceeded* | Firewall/SG (remote target), timeout | Allow 9100 from the Prometheus SG; check network/CPU load |
| Alertmanager fails: *no private IP address found* | Gossip needs an address | Add `--cluster.listen-address=` (empty) as in the unit file |
| Alertmanager target DOWN in Prometheus | Not started yet / port | `systemctl status alertmanager`; `curl localhost:9093/-/healthy` |
| Alert stays inactive though load is high | Expression returns nothing or threshold not crossed | Run the `expr` in the Query tab; lower the threshold; check labels |
| Alert stuck **pending** | Condition not true for the full `for:` duration | Keep the load running ≥ `for:` + a scrape or two |
| Firing in Prometheus but no webhook output | Prometheus not linked to Alertmanager; route/silence/inhibit; receiver down | `curl localhost:9090/api/v1/alertmanagers`; Alertmanager UI status; `systemctl status lab-webhook`; `amtool config routes test` |
| `promtool test rules` fails | Annotation text differs from the rule | Make `exp_annotations` identical to the rendered text |
| `stress-ng: command not found` | Not installed | `sudo apt install -y stress-ng` |
| Instance freezes during memory test | Too much memory requested | Reduce `--vm-bytes`; add timeout; reboot if needed |
| Disk alert never fires | Threshold above current usage | Create a larger file or lower the threshold; check `df -h /` |
| Cannot open `:9090/:9093` | Security group/firewall | Allow your IP; `sudo ss -ltnp | grep -E "9090|9093"` |

---

# 1️⃣2️⃣ EVIDENCE CHECKLIST: PRACTICAL 8 📸

```
☐ EC2 instance + security group (ports restricted to your IP)         ☐ node_exporter service active + /metrics output
☐ Prometheus service active + config file                              ☐ Status → Targets (node, prometheus, alertmanager UP)
☐ Baseline queries: CPU %, memory %, disk %, network bits/s             ☐ promtool check config/rules + promtool test rules SUCCESS
☐ Alert rules file; Alerts page (inactive/pending/firing states)        ☐ Alertmanager running; amtool check-config; routing test
☐ CPU stress: graph spike + FIRING alert + webhook message + RESOLVED   ☐ Memory, disk and instance-down scenarios (at least 3 scenarios)
☐ Silence demo (alert silenced, no webhook)                             ☐ Results table (times to pending/firing)
☐ Cleanup completed (fill file removed, stress stopped)
```

---

# 🅱️ PRACTICAL 9: GRAFANA DASHBOARDS

# 1️⃣3️⃣ STEP 10: INSTALL GRAFANA

Grafana's package signing key and repository layout have changed over time. **Copy the exact commands from the official page:** `https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/` ("Install from APT repository"). They look like this:

```bash
sudo apt-get install -y apt-transport-https wget gnupg
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc
echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana           # OSS package; Grafana's docs now also list "grafana-enterprise" as the recommended default edition (free, same features)
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server --no-pager      # active (running)
```

Verify:

```bash
curl -s localhost:3000/api/health      # {"database":"ok","version":"..."}  (the exact fields depend on the version)
sudo ss -ltnp | grep 3000
```

Browser: **`http://<EC2_PUBLIC_IP>:3000`** → login `admin` / `admin` → **set a new strong password**.

📸 Screenshot: Grafana login/home page showing the version.

---

# 1️⃣4️⃣ STEP 11: ADD PROMETHEUS AS A DATA SOURCE ⭐

```
☰ Menu → Connections → Data sources → Add data source → Prometheus
   Name:                   Prometheus
   Prometheus server URL:  http://localhost:9090
   Scrape interval:        15s        (matches Prometheus; important for $__rate_interval)
   (leave other defaults)
→ Save & test   →  "Successfully queried the Prometheus API."  ✅
```

Prove it works with **Explore**: Explore → data source Prometheus → run `up` and `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[$__rate_interval])) * 100)` → Run query.

📸 Screenshot: data source "Save & test" success message and an Explore graph.

---

# 1️⃣5️⃣ STEP 12: IMPORT DASHBOARDS (FAST START) ⭐

## 12.1 Community dashboard: Node Exporter Full

```
Dashboards → New → Import → Grafana.com dashboard URL or ID: 1860 → Load
→ Data source: Prometheus → Import
```

(Requires internet access from your browser/Grafana to grafana.com. If the ID has changed, search "Node Exporter Full" on grafana.com/grafana/dashboards.) Browse its sections: CPU, Memory, Network, Disk. If panels show *No data*, check the `job` variable matches your job name (`node`).

## 12.2 The supplied lab dashboard (`U4_infra-dashboard.json`)

```
Dashboards → New → Import → Upload dashboard JSON file → choose U4_infra-dashboard.json
→ Prometheus: select your data source → Import
```

It contains **27 panels** in sections **Overview, CPU, Memory, Storage, Network**, a `$instance` variable, shared crosshair and a 30-second refresh. All its queries were run against live Node Exporter data before delivery. Use it to **compare** with your own design in Step 13 and to see how thresholds, units and stacking are configured (open any panel → Edit).

📸 Screenshot: the imported community dashboard and the lab dashboard.

---

# 1️⃣6️⃣ STEP 13: DESIGN YOUR OWN INFRASTRUCTURE DASHBOARD ⭐⭐⭐

(The detailed panel table is in **Part 5, Section 13**; below is the hands-on sequence. Build at least **one panel per resource**; the complete version has 20+.)

## 13.1 Create dashboard and variable

```
Dashboards → New → New dashboard → (gear icon) Settings:
   Title: "My Infrastructure Dashboard"   Tags: node, infra   Time range: Last 3 hours   Auto refresh: 30s
Variables → Add variable:
   Type: Query   Name: instance   Label: Host   Data source: Prometheus
   Query: label_values(up{job="node"}, instance)   Refresh: On dashboard load
   Selection: ☑ Multi-value  ☑ Include All option
Save dashboard
```

## 13.2 Add the panels (Add → Visualization; use **Code** mode in the query editor)

| # | Visualization | Title | Query | Unit | Notes |
|---|---|---|---|---|---|
| 1 | **Gauge** | CPU now | `100 - (avg(rate(node_cpu_seconds_total{mode="idle", instance=~"$instance"}[$__rate_interval])) * 100)` | Percent (0-100) | min 0, max 100; thresholds green / **70** yellow / **90** red; Query type **Instant** |
| 2 | **Time series** | CPU usage (%) | `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle", instance=~"$instance"}[$__rate_interval])) * 100)` | Percent (0-100) | Legend `{{instance}}`; show threshold lines |
| 3 | **Time series** (stacked) | CPU by mode | `avg by (mode) (rate(node_cpu_seconds_total{mode!="idle", instance=~"$instance"}[$__rate_interval])) * 100` | Percent | Stack: Normal; legend `{{mode}}` |
| 4 | **Time series** | Load average | A `node_load1{instance=~"$instance"}`, B `node_load5{...}`, C `node_load15{...}` | none | Compare with number of cores |
| 5 | **Gauge** | Memory used | `(1 - sum(node_memory_MemAvailable_bytes{instance=~"$instance"}) / sum(node_memory_MemTotal_bytes{instance=~"$instance"})) * 100` | Percent | thresholds 80/90 |
| 6 | **Time series** (stacked) | Memory breakdown | A `node_memory_MemTotal_bytes - node_memory_MemFree_bytes - node_memory_Buffers_bytes - node_memory_Cached_bytes` (legend "Used"), B `node_memory_Buffers_bytes + node_memory_Cached_bytes` ("Cache"), C `node_memory_MemFree_bytes` ("Free"), all with `{instance=~"$instance"}` | bytes (IEC) | Stack: Normal |
| 7 | **Bar gauge** | Filesystem used % | `100 - (node_filesystem_avail_bytes{instance=~"$instance", fstype!~"tmpfs\|overlay\|squashfs\|ramfs"} / node_filesystem_size_bytes{instance=~"$instance", fstype!~"tmpfs\|overlay\|squashfs\|ramfs"} * 100)` | Percent | Instant; legend `{{mountpoint}}`; thresholds 75/90 (type a plain `\|` as `|` in Grafana) |
| 8 | **Time series** | Disk throughput | A `sum by (instance) (rate(node_disk_read_bytes_total{instance=~"$instance"}[$__rate_interval]))` ("read"), B `... node_disk_written_bytes_total ...` ("write") | bytes/sec (IEC) | |
| 9 | **Time series** | Disk utilization % | `rate(node_disk_io_time_seconds_total{instance=~"$instance"}[$__rate_interval]) * 100` | Percent | ~100% means saturated |
| 10 | **Time series** | Network in/out (bits/s) | A `sum by (instance) (rate(node_network_receive_bytes_total{instance=~"$instance", device!="lo"}[$__rate_interval])) * 8` ("in"), B `... transmit ...` ("out") | bits/sec (SI) | |
| 11 | **Stat** | Uptime | `min(time() - node_boot_time_seconds{instance=~"$instance"})` | seconds (s) | |
| 12 | **State timeline** | Target status | `up{job="node"}` | none | Value mappings: 1 → UP (green), 0 → DOWN (red) |

For each panel: set **Title**, **Unit**, **Min/Max**, **Thresholds**, **Legend**. Organize into **rows** (CPU, Memory, Storage, Network) via Add → Row. **Save** (give a version message).

## 13.3 Quality checklist ✅

```
☐ Every panel has a title and correct unit (percent 0-100 vs ratio 0-1!)     ☐ Thresholds colored consistently (green/yellow/red)
☐ Variable works (select one host, then All)                                  ☐ Uses $__rate_interval (or a window ≥ 4× scrape interval)
☐ Rows follow the USE method (utilization → saturation → errors)             ☐ Top-left shows the most important health stats
☐ Dashboard time range/refresh sensible; shared crosshair enabled            ☐ Text panel with purpose/owner (optional)
```

📸 Screenshot: your finished dashboard (full page) and one panel's edit view showing query, unit and thresholds.

---

# 1️⃣7️⃣ STEP 14: GENERATE LOAD AND ANALYZE PERFORMANCE TRENDS ⭐⭐⭐

Set the dashboard to **Last 1 hour** and refresh **10 s**. Create the following events one at a time, **wait 1-2 minutes** between them, and note the **clock time** of each (use the same time zone as the dashboard).

| # | Action (run on the server) | Expected visible effect |
|---|---|---|
| 0 | Idle baseline (10 minutes) | Low CPU, stable memory, flat disk, tiny network |
| 1 | `stress-ng --cpu 0 --timeout 180s` | CPU % → ~100% within ~30 s; "CPU by mode" shows **user** dominating; **load average** rises with a delay (load1 first, then load5), per-core load > 1 |
| 2 | `stress-ng --vm 1 --vm-bytes 1G --vm-keep --timeout 180s` | Memory used % jumps; breakdown "Used (apps)" grows; **available** shrinks; swap normally unchanged |
| 3 | `dd if=/dev/zero of=/var/tmp/io.test bs=1M count=2048 oflag=direct; sync` then `dd if=/var/tmp/io.test of=/dev/null bs=1M iflag=direct; rm /var/tmp/io.test` | Disk **write** then **read** throughput spikes; disk utilization % rises; iowait may rise; free space dips then recovers |
| 4 | `sudo fallocate -l 7G /var/tmp/fill.img` (wait 2 min) then `sudo rm /var/tmp/fill.img` | Filesystem used % steps up and back; free-space panel shows the drop; forecast panel reacts |
| 5 | `for i in 1 2 3; do curl -sL -o /dev/null https://github.com/prometheus/prometheus/releases/download/v${PROM_VER}/prometheus-${PROM_VER}.linux-amd64.tar.gz; done` | **Network inbound** bits/s spike during downloads (loopback `lo` is excluded, so local traffic doesn't count) |
| 6 | `sudo systemctl stop node_exporter` for 60 s, then start | Gaps in graphs; **Target status** timeline shows DOWN then UP; the Targets-down stat turns red |

(If `PROM_VER` isn't set in your new shell: `PROM_VER=3.13.0`.)

## 14.1 Trend analysis table (fill it in) ✍️

| Observation | Value (from your dashboard) |
|---|---|
| Baseline CPU / memory / disk / network | |
| Peak CPU % and time | |
| CPU mode that dominated during stress | |
| Time for load1 to peak vs CPU % to peak (lag) | |
| Memory used % before / during stress; available memory low point | |
| Disk write MB/s during `dd`; disk utilization % peak | |
| Root filesystem used % before / during / after the 7 GiB file | |
| Peak inbound network Mbit/s during downloads | |
| Alerts that fired during the tests (link with Step 9) | |

## 14.2 Analysis paragraph (example wording; use your own numbers)

```
"During the CPU stress test between 10:31 and 10:36 the CPU usage rose from a baseline of about 2% to 100%, dominated by user-mode
time, while the 1-minute load average climbed above 2 on a 2-core instance after about a minute, confirming CPU saturation. Memory
usage increased from 24% to 71% when 1 GiB was allocated and returned to baseline afterwards. The dd test produced sustained disk
writes of about 130 MiB/s with disk utilization near 100% (IOPS-limited gp3 volume), and the 7 GiB file moved root usage from 14% to
52%. Inbound traffic peaked near 300 Mbit/s during package downloads. The InstanceDown test appeared as a gap in the metrics and a red
segment in the status timeline. These patterns match expected behavior and show how the dashboard supports capacity planning and
troubleshooting."
```

(Numbers above are illustrative. Report **what you observed**.)

## 14.3 Compare time ranges and trends

```
• Zoom to the stress window (drag on a graph) and note start/end values; use "Last 6 hours" to see the whole session
• Add a second query with `offset 1d` on the CPU panel to compare with yesterday (after the instance has >24 h of data)
• Add an annotation (Ctrl/Cmd+click on a graph) for each test: "CPU stress", "disk fill", "network download"
• Optionally add the Prometheus query  ALERTS{alertstate="firing"}  as a table panel to see which alerts were active
```

📸 Screenshots: dashboard during CPU stress; during disk write; during the network download; the time range of the full session with annotations.

---

# 1️⃣8️⃣ OPTIONAL: A GRAFANA ALERT RULE (COMPARE WITH PROMETHEUS ALERTS) ⭐

```
Alerting → Contact points → Add contact point: type Webhook, URL http://localhost:5001/  → Test (appears in lab-webhook output)
Alerting → Alert rules → New alert rule:
   Name: GrafanaHighCPU   Query A (Prometheus): 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
   Condition: Reduce (Last) → Threshold IS ABOVE 60
   Folder: Lab   Evaluation group: lab (every 1m)   Pending period: 1m   Label: severity=warning
   Notification: default policy → contact point you created
Run CPU stress → state goes Normal → Pending → Alerting → webhook receives the notification
```

Compare (for the report): Prometheus rules (code, independent of Grafana, routed by Alertmanager) vs Grafana alerting (UI-managed, many data sources).

---

# 1️⃣9️⃣ SAVE YOUR WORK: EXPORT AND PROVISION 📦

```
Export:  Dashboard → Share/Export → Export as JSON (tick "Export for sharing externally") → save to Git
Provision (optional): copy the JSON to /var/lib/grafana/dashboards/ and add provisioning files from Part 5 §17,
                      then  sudo systemctl restart grafana-server
```

Forgot the Grafana admin password? `sudo grafana cli admin reset-admin-password '<NewStrongPassword>'`.

---

# 2️⃣0️⃣ TROUBLESHOOTING (PRACTICAL 9) 🛠️

| Symptom | Likely cause | Fix |
|---|---|---|
| Can't open `:3000` | Grafana not running / SG | `systemctl status grafana-server`; open 3000 to your IP; `journalctl -u grafana-server` |
| `apt update` signature error for Grafana repo | Key file name changed | Use the current key URL from the Grafana install docs |
| Data source test: *connection refused* | Wrong URL/port; Prometheus down | `http://localhost:9090` (same host); `curl localhost:9090/-/healthy` |
| Panels show **No data** | Wrong `job` label; target down; empty variable; time range before data existed | Check `up`; use `job="node"`; verify `$instance` has values; set time range to *Last 1 hour* |
| Variable dropdown empty | Wrong `label_values` query/data source | Test in Explore; use `label_values(up{job="node"}, instance)` |
| "All" shows nothing | Used `=` instead of `=~` | `instance=~"$instance"` |
| Values 100× off | Percent (0-100) vs ratio (0-1) unit | Match unit to the query |
| Graph flat/spiky | `rate` window vs scrape interval | Use `$__rate_interval`; set data source Scrape interval 15s |
| Imported dashboard N/A | Job names or data source mismatch | Edit variables; select the correct data source on import |
| Memory/Disk panels empty for "swap" | Host has no swap (`SwapTotal = 0` → division gives NaN) | Expected; ignore or hide the panel |
| Network panel flat during tests | Traffic on `lo` only | Generate **external** traffic (download from the internet) |
| Imported JSON error | Wrong file/data source input | Re-import and choose the Prometheus data source when prompted |

---

# 2️⃣1️⃣ EVIDENCE CHECKLIST: PRACTICAL 9 📸

```
☐ Grafana service active + login page/home (version)                    ☐ Data source configuration + "Successfully queried the Prometheus API"
☐ Explore query graph                                                     ☐ Imported community dashboard (Node Exporter Full)
☐ Your dashboard: overview + CPU + memory + storage + network panels      ☐ Panel edit view (query, unit, thresholds), variable dropdown
☐ Dashboard during CPU stress / disk write / network download             ☐ Trend analysis table and analysis paragraph
☐ Optional: Grafana alert rule + webhook notification                     ☐ Exported dashboard JSON (kept in Git)
☐ Cleanup completed
```

---

# 2️⃣2️⃣ CLEANUP 🧹 (MANDATORY)

```bash
sudo rm -f /var/tmp/fill.img /var/tmp/io.test
pkill stress-ng 2>/dev/null
# optional: stop services
sudo systemctl disable --now grafana-server prometheus alertmanager node_exporter lab-webhook
```

```
☐ Terminate (or stop) the EC2 instance(s) ; release any Elastic IP ; delete security groups you created
☐ Delete the key pair if unused ; revoke any Gmail app password/Slack webhook you created
☐ Check Billing/Cost Explorer next day for stray charges
☐ Keep your configs, dashboard JSON and report screenshots in Git
```

---

# 2️⃣3️⃣ REPORT TEMPLATES 📝

```
PRACTICAL 8 REPORT
Aim            : (copy the syllabus line)
Requirements   : AWS EC2 Ubuntu, Prometheus, Node Exporter, Alertmanager, webhook receiver, stress-ng
Theory         : monitoring vs observability, Prometheus architecture, metric types, Node Exporter, PromQL, alert rules, Alertmanager
Architecture   : diagram from §0
Procedure      : steps 1-9 with commands (summarize)
Config files   : prometheus.yml, alerts.yml, alertmanager.yml (with explanation of each block)
Output         : screenshots (targets, queries, alert states, webhook output, silences)
Results table  : times to pending/firing and resolution per scenario
Result/Conclusion/Precautions (restricted ports, cleanup)

PRACTICAL 9 REPORT
Aim            : (copy the syllabus line)
Requirements   : Grafana, Prometheus data source, Node Exporter data, load tools
Theory         : Grafana architecture, data sources, panels, variables, USE method, dashboard design
Procedure      : steps 10-14 (install, data source, import, design, load tests)
Dashboard      : screenshots of every row; panel queries and units table
Trend analysis : filled table + paragraph (with timestamps)
Result/Conclusion/Precautions
```

---

# 🎯 PRACTICAL VIVA QUESTIONS

**Q1. Why use a dedicated non-login user for Prometheus and Node Exporter?** Least privilege: if the service is compromised the attacker doesn't get a shell or root access.

**Q2. What does Node Exporter do and on which port?** Exposes host metrics (CPU, memory, disk, network) at `/metrics` on port 9100.

**Q3. How does Prometheus get the data?** It pulls (scrapes) `/metrics` over HTTP every `scrape_interval` as defined in `scrape_configs`.

**Q4. How do you check that scraping works?** Status → Targets shows UP, and the query `up` returns 1.

**Q5. Write the PromQL for CPU usage and memory usage.** `100 - avg(rate(node_cpu_seconds_total{mode="idle"}[1m]))*100` and `(1 - node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes)*100`.

**Q6. What is the purpose of `for:` in your alert rule?** To require the condition to hold for a period, avoiding false alerts from short spikes (pending → firing).

**Q7. How did you validate alert generation?** Generated real load with stress-ng/fallocate/stopping the exporter and observed the alert in Prometheus (pending/firing), Alertmanager UI and the webhook receiver output, plus a resolved message afterwards.

**Q8. What does Alertmanager do that Prometheus doesn't?** Groups, deduplicates, routes, silences/inhibits and delivers notifications.

**Q9. What is an inhibition rule and what is a silence?** Inhibition automatically suppresses target alerts when a source alert fires (e.g., InstanceDown); a silence is a manual time-limited mute.

**Q10. Why use `promtool`?** To validate configs/rules and unit-test alert rules before deploying.

**Q11. How did you connect Grafana to Prometheus?** Added a Prometheus data source with URL `http://localhost:9090` and scrape interval 15s, then tested it.

**Q12. What is `$__rate_interval` and why use it?** A Grafana variable that picks a suitable `rate()` window from the scrape interval and zoom level.

**Q13. Why `MemAvailable` instead of `MemFree`?** MemAvailable includes reclaimable cache, so it represents memory actually available to applications.

**Q14. How do variables make a dashboard reusable?** The `$instance` dropdown (from `label_values`) filters queries via `instance=~"$instance"`, so one dashboard serves all hosts.

**Q15. How would you extend this lab for production?** Real thresholds with longer `for`, HA Prometheus/Alertmanager, remote storage (Thanos/Mimir), TLS/auth, service discovery, SLO burn-rate alerts, dashboards and rules in Git.

---

# ✅ QUICK CHECKLISTS

**Practical 8:** EC2 + SG → users → Node Exporter (systemd) → Prometheus (systemd) → `prometheus.yml` + `promtool check` → UI Targets UP → baseline PromQL (CPU, memory, disk, network) → Alertmanager + webhook receiver → `alerts.yml` + `promtool check/test` → stress tests (CPU, memory, disk, stop exporter) → observe pending/firing/webhook/resolved → silence demo → evidence → cleanup.

**Practical 9:** Install Grafana → login + new password → Prometheus data source (Save & test) → import Node Exporter Full (1860) + lab JSON → build own dashboard (variable + CPU, memory, storage, network panels with units and thresholds) → generate load (CPU, memory, disk I/O, disk fill, network, exporter stop) → analyze trends (table + paragraph) → optional Grafana alert → export JSON → cleanup.

---

## 🫡 PART 8 STATUS

Both Unit IV practicals are fully scripted with tested configs, real alert-validation experiments, a supplied importable dashboard, trend-analysis templates, troubleshooting and cleanup.

**STOP here.** Last file:

> **🏆 GRAND CONSOLIDATION: Unit IV full revision, comparison tables, PromQL cheat sheets, 30 viva Qs, traps, master sheet**
