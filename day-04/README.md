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

---

## 🐳 Docker Service

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

## 🔧 Add to Prometheus

```yaml
- job_name: "otel-collector"
  static_configs:
    - targets: ["otel-collector:8889"]
```

---

# 🧪 3. Send Test Data

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

---

# 🚨 4. Prometheus Alerting

## 📄 alert-rules.yml

### Example: High CPU

```yaml
- alert: HighCPUUsage
  expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "High CPU usage"
```

---

## 🔧 Load Rules

```yaml
rule_files:
  - /etc/prometheus/alert-rules.yml
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

## 🔹 Create Contact Point

* Email / Slack

---

## 🔹 Create Alert Rule

Example:

```promql
container_memory_usage_bytes{name=~".*notes-app.*"} / 1024 / 1024
```

Trigger:

```
> 100 MB for 2 minutes
```

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
