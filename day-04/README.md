# 🔍 Day 04 – OpenTelemetry & Alerting

Welcome to **Day 04** of the *DevOps Observability Stack*.

In this module, we complete the **three pillars of observability** by adding:

* **Tracing** via OpenTelemetry
* **Alerting** via Prometheus & Grafana

---

# 🚀 What You Will Learn

* OpenTelemetry architecture (Receivers, Processors, Exporters)
* OTLP protocol (gRPC & HTTP)
* Sending traces and metrics to OTEL Collector
* Prometheus alerting rules
* Grafana alerting & notifications
* Full observability pipeline integration

---

# 🏗️ Architecture Overview

## 📊 Metrics Pipeline

```
Node Exporter → Prometheus → Grafana
cAdvisor      → Prometheus → Grafana
OTEL Metrics  → Prometheus → Grafana
```

## 📜 Logs Pipeline

```
Docker → Promtail → Loki → Grafana
```

## 🔍 Traces Pipeline

```
App / curl → OTEL Collector → Debug (→ future: Jaeger/Tempo)
```

---

# 🧠 1. OpenTelemetry Concepts

## 🔹 What is OpenTelemetry?

* Vendor-neutral observability framework
* Collects: **metrics, logs, traces**
* Sends to backends (Prometheus, Loki, Jaeger, etc.)

---

## 🔹 OTEL Collector Pipeline

```
Receivers → Processors → Exporters
```

| Component | Role                     |
| --------- | ------------------------ |
| Receiver  | Accepts telemetry (OTLP) |
| Processor | Transforms/batches data  |
| Exporter  | Sends to backend         |

---

## 🔹 OTLP Protocol

* Standard telemetry format
* Ports:

  * **4317 → gRPC**
  * **4318 → HTTP**

---

## 🔹 Distributed Traces

* A **trace** = full request journey
* A **span** = one step in that journey

Example:

```
User → API → Auth → DB
```

---

# ⚙️ 2. OTEL Collector Setup

Create the collector configuration:

```bash
mkdir -p otel-collector
```
## 📄 otel-collector-config.yml

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch:

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
  debug:
    verbosity: detailed

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]

    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]

    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
```
What this config does:

* Receivers: Accepts OTLP data via gRPC (4317) and HTTP (4318)
* Processors: Batches data before exporting (reduces overhead)
* Exporters:
    * Metrics go to a Prometheus-compatible endpoint on port 8889 (Prometheus scrapes this)
    * Traces and logs go to debug output (console) -- in production you would send these to Jaeger or Tempo
---

## 🐳 Docker Service (docker-compose.yml)

```yaml
otel-collector:
  image: otel/opentelemetry-collector-contrib:latest
  container_name: otel-collector
  ports:
    - "4317:4317"
    - "4318:4318"
    - "8889:8889"
  volumes:
    - ./otel-collector/otel-collector-config.yml:/etc/otelcol-contrib/config.yaml
  restart: unless-stopped
```

---

## 🔧 Add to Prometheus (prometheus.yml)

```yaml
- job_name: "otel-collector"
  static_configs:
    - targets: ["otel-collector:8889"]
```
Restart everything:
```bash
docker compose up -d
```
Verify the collector is running:

```bash
docker logs otel-collector 2>&1 | tail -5
```
Check Prometheus Targets -- you should now see otel-collector as UP.

---

# 🧪 3. Send Test Data

Send a sample OTLP trace using curl:

```bash
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{
          "key": "service.name",
          "value": { "stringValue": "my-test-service" }
        }]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "5b8efff798038103d269b633813fc60c",
          "spanId": "eee19b7ec3c1b174",
          "name": "test-span",
          "kind": 1,
          "startTimeUnixNano": "1544712660000000000",
          "endTimeUnixNano": "1544712661000000000",
          "attributes": [{
            "key": "http.method",
            "value": { "stringValue": "GET" }
          },
          {
            "key": "http.status_code",
            "value": { "intValue": "200" }
          }]
        }]
      }]
    }]
  }'
```

Check the collector debug output to see the trace:
```bash
docker logs otel-collector 2>&1 | grep -A 10 "test-span"
```

You should see the span details printed to the console. In a production setup, you would send these to a trace backend like Jaeger or Grafana Tempo for storage and visualization.

**Send OTLP metrics too:**
```bash
curl -X POST http://localhost:4318/v1/metrics \
  -H "Content-Type: application/json" \
  -d '{
    "resourceMetrics": [{
      "resource": {
        "attributes": [{
          "key": "service.name",
          "value": { "stringValue": "my-test-service" }
        }]
      },
      "scopeMetrics": [{
        "metrics": [{
          "name": "test_requests_total",
          "sum": {
            "dataPoints": [{
              "asInt": "42",
              "startTimeUnixNano": "1544712660000000000",
              "timeUnixNano": "1544712661000000000"
            }],
            "aggregationTemporality": 2,
            "isMonotonic": true
          }
        }]
      }]
    }]
  }'
```
## 🔹 Send Trace

```bash
curl -X POST http://localhost:4318/v1/traces ...
```

Check:

```bash
docker logs otel-collector | grep test-span
```

---

## 🔹 Send Metrics

```bash
curl -X POST http://localhost:4318/v1/metrics ...
```

Query in Prometheus:

```promql
test_requests_total
```
The metric traveled: your curl command -> OTEL Collector (OTLP receiver) -> Prometheus exporter -> Prometheus scraped it. This is how OTEL bridges different telemetry formats.
---

# 🚨 4. Prometheus Alerting

## 📄 alert-rules.yml
Alerts notify you when something is wrong. Prometheus evaluates alerting rules and fires alerts when conditions are met.

### Create an alerting rules file alert-rules.yml:

```yaml
groups:
  - name: system-alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage has been above 80% for more than 2 minutes. Current value: {{ $value }}%"

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "Memory usage is above 85%. Current value: {{ $value }}%"

      - alert: ContainerDown
        expr: absent(container_last_seen{name="notes-app"})
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container is down"
          description: "The notes-app container has not been seen for over 1 minute"

      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Scrape target is down"
          description: "{{ $labels.job }} target {{ $labels.instance }} is unreachable"

      - alert: HighDiskUsage
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space running low"
          description: "Root filesystem usage is above 90%. Current value: {{ $value }}%"
```
What each alert does:

* expr -- the PromQL condition that triggers the alert
* for -- how long the condition must be true before firing (avoids flapping)
* labels -- metadata for routing (severity: warning vs critical)
* annotations -- human-readable description

Update prometheus.yml to load the rules:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/alert-rules.yml

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: "otel-collector"
    static_configs:
      - targets: ["otel-collector:8889"]
```
---

## 🔧 Load Rules

```yaml
rule_files:
  - /etc/prometheus/alert-rules.yml
```
Mount the rules file in docker-compose.yml under the Prometheus service:
```yaml
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert-rules.yml:/etc/prometheus/alert-rules.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: unless-stopped
```

Restart Prometheus:
```bash
docker compose up -d prometheus
```

Check the rules in the Prometheus UI: go to Status > Rules. You should see all five alert rules listed.

Go to Alerts -- they should be in `inactive` state (green). If any condition is true, the alert moves to `pending`, then `firing` after the `for` duration.

**Test it:** Stop the notes-app container and watch the `TargetDown` alert fire:
```bash
docker compose stop notes-app
```

Wait 1-2 minutes, then check Alerts in the Prometheus UI. Start it back up when done:
```bash
docker compose start notes-app
```
---

## 🔍 Verify

Prometheus UI:

```
Status → Rules
Alerts → check state
```

---

# 📊 5. Grafana Alerting

Grafana can also evaluate alerts and send notifications to Slack, email, PagerDuty, and more.

1. **Create a contact point:**
   - Go to Alerting > Contact points > Add contact point
   - Name: "DevOps Team"
   - Integration: Choose email (or Slack webhook if you have one)
   - For email: just enter your email address
   - Save

2. **Create an alert rule in Grafana:**
   - Go to Alerting > Alert rules > New alert rule
   - Name: "High Container Memory"
   - Query: `container_memory_usage_bytes{name="notes-app"} / 1024 / 1024`
   - Condition: IS ABOVE 100 (fire if container uses more than 100MB)
   - Evaluation: every 1m, for 2m
   - Add label: severity = warning
   - Link to the "DevOps Team" contact point
   - Save

3. **Create a notification policy:**
   - Go to Alerting > Notification policies
   - Set the default contact point to "DevOps Team"
   - Add a nested policy: match label `severity=critical` -> route to a different contact point (or the same one with different settings)

4. **View alert state:**
   - Go to Alerting > Alert rules
   - You should see your rule in Normal, Pending, or Firing state


---

# ⚖️ Prometheus vs Grafana Alerts

| Feature     | Prometheus   | Grafana              |
| ----------- | ------------ | -------------------- |
| Evaluation  | Built-in     | External             |
| Best for    | Infra alerts | Visualization alerts |
| Flexibility | High         | UI-driven            |
| Integration | Alertmanager | Multiple channels    |

👉 Use:

* **Prometheus → core system alerts**
* **Grafana → UI-driven alerts**

---

# 🧠 6. Full Stack Summary

| Service        | Port           | Purpose           |
| -------------- | -------------- | ----------------- |
| Prometheus     | 9090           | Metrics           |
| Node Exporter  | 9100           | Host metrics      |
| cAdvisor       | 8080           | Container metrics |
| Grafana        | 3000           | Dashboards        |
| Loki           | 3100           | Logs              |
| Promtail       | 9080           | Log collection    |
| OTEL Collector | 4317/4318/8889 | Telemetry         |
| Notes App      | 8000           | Sample app        |

---

# 🧾 Key Takeaways

* Observability = **Metrics + Logs + Traces**
* OTEL = **unified telemetry pipeline**
* Prometheus = **metrics + alert engine**
* Grafana = **visual + alert UI**
* Alerts = **proactive monitoring**

---

# 🔥 Final Result

You now have a **production-style observability stack** with:

✔ Metrics
✔ Logs
✔ Traces
✔ Alerting

---

# 📌 Author

**Shibnath**
DevOps | Cloud | Observability Engineer 🚀

---
