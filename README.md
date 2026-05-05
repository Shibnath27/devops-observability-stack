# 🚀 DevOps Observability Stack (Day 01-05)

A complete, production-style observability platform built step-by-step using **Prometheus, Grafana, Loki, and OpenTelemetry**.

This repository demonstrates how to design, deploy, and validate **metrics, logs, and traces pipelines**—the three pillars of modern observability.

---

# 🎯 Project Goal

Build an end-to-end observability system that answers:

* What is broken? → **Metrics**
* Why is it broken? → **Logs**
* Where did it break? → **Traces**

---

# 🧠 Architecture Overview

```text
                        ┌────────────────────┐
                        │     Grafana        │
                        │ Dashboards + Alerts│
                        └─────────┬──────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼

   ┌──────────────┐        ┌──────────────┐         ┌──────────────┐
   │ Prometheus   │        │    Loki      │         │ OTEL Collector│
   │ (Metrics DB) │        │ (Logs Store) │         │ (Traces Pipe) │
   └──────┬───────┘        └──────┬───────┘         └──────┬────────┘
          │                       │                        │
          ▼                       ▼                        ▼

 ┌──────────────┐        ┌──────────────┐         ┌──────────────┐
 │ NodeExporter │        │   Promtail   │         │   OTLP Input │
 │ (Host Metrics)│       │ (Log Agent)  │         │ (App / curl) │
 └──────────────┘        └──────────────┘         └──────────────┘

 ┌──────────────┐
 │  cAdvisor    │
 │ (Containers) │
 └──────────────┘

 ┌──────────────┐
 │  Notes App   │
 │ Demo Workload│
 └──────────────┘
```

---

# 🔁 Data Flow (End-to-End)

## 📊 Metrics Pipeline

```text
Node Exporter / cAdvisor / OTEL
        ↓
   Prometheus (scrape)
        ↓
     Grafana
        ↓
     Alerts
```

## 📜 Logs Pipeline

```text
Docker Containers
        ↓
     Promtail
        ↓
       Loki
        ↓
     Grafana
```

## 🔍 Traces Pipeline

```text
Application / curl
        ↓
  OTEL Collector
        ↓
 Debug Output (future: Tempo)
```

---

# ⚙️ Workflow (How Everything Works)

### 1. Data Collection

* Node Exporter → host metrics
* cAdvisor → container metrics
* Promtail → container logs
* OTEL → traces + custom metrics

### 2. Data Aggregation

* Prometheus → metrics storage (TSDB)
* Loki → logs (label-based indexing)
* OTEL Collector → telemetry routing

### 3. Visualization

* Grafana dashboards (auto-provisioned)
* Logs + metrics correlation
* Explore mode for debugging

### 4. Alerting

* Prometheus rules (infrastructure alerts)
* Grafana alerts (application-level alerts)

---

# 📁 Repository Structure

```text
devops-observability-stack/
│
├── day-01/   # Prometheus basics
├── day-02/   # Exporters + Grafana dashboards
├── day-03/   # Loki + Promtail (logs)
├── day-04/   # OTEL + Alerting
│
└── day-05-observability-project/   # Full integrated system
    ├── docker-compose.yml
    ├── prometheus.yml
    ├── alert-rules.yml
    ├── grafana/
    ├── loki/
    ├── promtail/
    ├── otel-collector/
    └── notes-app/
```

---

# ⚡ Quick Start (Run Full Stack)

```bash
cd day-05-observability-project
docker compose up -d
docker compose ps
```

---

# 🌐 Access Points

| Service    | URL                   | Purpose             |
| ---------- | --------------------- | ------------------- |
| Grafana    | http://localhost:3000 | Dashboards & alerts |
| Prometheus | http://localhost:9090 | Metrics             |
| Loki       | http://localhost:3100 | Logs backend        |
| cAdvisor   | http://localhost:8080 | Container metrics   |
| Notes App  | http://localhost:8000 | Demo app            |

---

# ✅ Validation Guide

## Metrics Check

```promql
up
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
topk(3, container_memory_usage_bytes{name!=""})
```

## Logs Check

```logql
{job="docker"}
{container_name="notes-app"} |= "GET"
```

## Traces Check

```bash
docker logs otel-collector
```

---

# 📊 Key Dashboards

* System Health (CPU, Memory, Disk)
* Container Metrics (CPU, Memory usage)
* Logs (Live container logs)
* Error Rate & Log Volume
* Service Health (scrape latency, ingestion)

---

# 🔔 Alerting Strategy

### Prometheus Alerts

* High CPU (>80%)
* High Memory (>75%)
* Disk Usage (>90%)
* Target Down
* Container Down

### Grafana Alerts

* Container memory spikes
* Application-level anomalies

---

# 📈 What Makes This Project Strong

✔ Full observability stack (not just one tool)
✔ Real data pipelines (metrics, logs, traces)
✔ Alerting + dashboards
✔ Docker-based reproducible setup
✔ Production-style architecture

---

# ⚖️ vs Managed Tools

| Feature     | This Stack | Cloud (Datadog / CloudWatch) |
| ----------- | ---------- | ---------------------------- |
| Cost        | Free       | Paid                         |
| Control     | Full       | Limited                      |
| Setup       | Manual     | Easy                         |
| Flexibility | High       | Medium                       |

---

# 🚀 Production Enhancements

* Alertmanager (Slack / PagerDuty integration)
* Grafana Tempo (trace storage)
* TLS/HTTPS security
* Authentication (Grafana + Prometheus)
* S3/GCS storage for logs & metrics
* HA (multi-node Prometheus + Loki)

---

# 🧠 Key Learnings

| Day | Focus                   |
| --- | ----------------------- |
| 01  | Prometheus + Metrics    |
| 02  | Exporters + Grafana     |
| 03  | Logs (Loki + Promtail)  |
| 04  | Traces + Alerting       |
| 05  | Full System Integration |

---

# 🧹 Cleanup

```bash
docker compose down
docker compose down -v
```

---

# 👨‍💻 Author

Built by Shibnath as part of a structured DevOps learning journey focused on real-world observability systems.

---
