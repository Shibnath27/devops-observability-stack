# 📊 Day 01 – Observability Fundamentals & Prometheus Setup

Welcome to **Day 01** of the *Observability for DevOps* journey.

In this module, we move beyond infrastructure setup and focus on **system visibility** — understanding what is happening inside your systems in real-time using **Prometheus**.

---

# 🚀 What You Will Learn

* Difference between **Monitoring vs Observability**
* The **Three Pillars of Observability**
* Core **Prometheus Concepts**
* Writing basic **PromQL queries**
* Running Prometheus using **Docker Compose**
* Scraping metrics from:

  * Prometheus itself
  * A sample application

---

# 🧠 1. Observability vs Monitoring

### ✅ Monitoring

* Detects **when something is wrong**
* Based on predefined thresholds and alerts
* Example: CPU > 90%

### ✅ Observability

* Helps understand **why something is wrong**
* Enables deep debugging using data correlation

👉 Key Idea:

> Monitoring tells you *there is a problem*
> Observability tells you *where and why*

---

# 🔺 2. Three Pillars of Observability

### 📈 Metrics

* Numerical data over time
* Examples:

  * CPU usage
  * Request rate
  * Error rate

**Tools:** Prometheus, CloudWatch, Datadog

---

### 📜 Logs

* Timestamped event records
* Examples:

  * Application errors
  * Debug messages

**Tools:** Loki, ELK Stack, Fluentd

---

### 🔍 Traces

* Tracks request flow across services

**Tools:** OpenTelemetry, Jaeger, Zipkin

---

# 🏗️ 3. Architecture Overview

```
[Application] --> Metrics --> Prometheus --> Grafana
[Application] --> Logs --> Promtail --> Loki --> Grafana
[Application] --> Traces --> OTEL Collector --> Grafana

[Host] --> Node Exporter --> Prometheus
[Docker] --> cAdvisor --> Prometheus
```

---

# 🐳 4. Project Setup

## 📁 Directory Structure

```
devops-observability-stack/
└── day-01/
    ├── docker-compose.yml
    └── prometheus.yml
```

---

# ⚙️ 5. Prometheus Configuration

## Create a project directory for this entire observability block -- you will keep adding to it over the next 5 days.

```bash
mkdir observability-stack && cd observability-stack
```

## 📄 prometheus.yml

Create a prometheus.yml configuration file:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "notes-app"
    static_configs:
      - targets: ["notes-app:8000"]
```

---

# 🐳 6. Docker Compose Setup

## 📄 docker-compose.yml

Create a docker-compose.yml to run Prometheus:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: unless-stopped

  notes-app:
    image: trainwithshubham/notes-app:latest
    container_name: notes-app
    ports:
      - "8000:8000"
    restart: unless-stopped

volumes:
  prometheus_data:
```

---

# ▶️ 7. Run the Stack

```bash
docker compose up -d
```

---

# 🌐 8. Access Prometheus

Open in browser:

```
http://localhost:9090
```

---

# ✅ 9. Verify Targets

Go to:

```
Status → Targets
```

You should see:

* ✅ prometheus (UP)
* ✅ notes-app (UP)

---

# 🔍 10. Prometheus Concepts

### 🔹 Counter

* Only increases
* Example: total requests

### 🔹 Gauge

* Can increase/decrease
* Example: memory usage

👉 Difference:

* Counter = cumulative metric
* Gauge = current state metric

---

### 🔹 Labels

Adds dimensions:

```
http_requests_total{method="GET", status="200"}
```

---

### 🔹 Time Series

Combination of:

```
Metric name + Labels
```

---

# 📊 11. PromQL Basics

### 🔹 Check target health

```
up
```

---

### 🔹 Total metrics count

```
count({__name__=~".+"})
```

---

### 🔹 Memory usage (MB)

```
process_resident_memory_bytes / 1024 / 1024
```

---

### 🔹 HTTP requests

```
prometheus_http_requests_total
```

---

### 🔹 Rate (most important)

```
rate(prometheus_http_requests_total[5m])
```

---

### 🔹 Aggregation

```
sum(rate(prometheus_http_requests_total[5m]))
```

---

### 🔹 Filter by status code

```
prometheus_http_requests_total{code="200"}
```

---

### 🔹 ❗ Challenge Query (Non-200 errors)

```
rate(prometheus_http_requests_total{code!="200"}[5m])
```

---

# 🧪 12. Generate Traffic

```bash
curl http://localhost:8000
curl http://localhost:8000
curl http://localhost:8000
```

---

# 💾 13. Data Storage & Retention

### Check storage usage

```bash
docker exec prometheus du -sh /prometheus
```

---

### Default Behavior

* Stores data in **TSDB**
* Default retention: **15 days**

---

### Customize retention

```yaml
command:
  - '--storage.tsdb.retention.time=30d'
  - '--storage.tsdb.retention.size=1GB'
```

---

### ⚠️ Important Concepts

* When retention exceeds:
  → Old data is automatically deleted

* Why volumes matter:
  → Prevents data loss when container restarts

---

# ⚠️ Troubleshooting

### Target DOWN?

Check:

* Container running?
* Correct port?
* Same Docker network?

---

### Config not updating?

Restart:

```bash
docker compose restart
```

---

# 🧾 Key Takeaways

* Prometheus uses a **pull-based model**
* `up` metric = health indicator
* Always use:

```
sum(rate(...))
```

NOT:

```
rate(sum(...))
```

---

# 🔥 What's Next?

In upcoming days, you will integrate:

* Node Exporter (Host metrics)
* cAdvisor (Container metrics)
* Loki (Logs)
* Grafana (Visualization)
* OpenTelemetry (Tracing)

---

# 📌 Author

**Shibnath**
DevOps | Cloud | Observability Engineer (in progress 🚀)

---
