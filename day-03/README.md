# 📜 Day 03 – Log Management with Loki & Promtail

Welcome to **Day 03** of the *DevOps Observability Stack*.

In this module, we implement the **second pillar of observability: Logs** using **Loki** and **Promtail**, and integrate them into **Grafana** alongside metrics.

---

# 🚀 What You Will Learn

* Log aggregation using **Loki**
* Log collection using **Promtail**
* Querying logs using **LogQL**
* Correlating **metrics + logs** in Grafana
* Building a complete observability pipeline

---

# 🏗️ Architecture Overview

```
[Docker Containers]
        |
        v
[Promtail] ---> [Loki] ---> [Grafana]
                          ↑
                    [Prometheus]
```

---

# 🧠 1. Why Loki?

Traditional logging systems (ELK) index **full log content** → expensive

### Loki Approach:

* Index **only labels**
* Store logs as compressed chunks

---

## ⚖️ Trade-off

| Advantage              | Trade-off                              |
| ---------------------- | -------------------------------------- |
| Low storage cost       | Slower full-text search                |
| Simpler setup          | Requires good labeling                 |
| Prometheus-like design | Less powerful than ELK for deep search |

👉 Loki = **efficient, scalable logging for DevOps**

---

# 📁 Project Structure

```
day-03/
├── docker-compose.yml
├── loki/
│   └── loki-config.yml
└── promtail/
    └── promtail-config.yml
```

---

# ⚙️ 2. Loki Setup

Create the Loki configuration file.

```bash
mkdir -p loki
```
## 📄 loki/loki-config.yml

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory
  replication_factor: 1
  path_prefix: /loki

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  filesystem:
    directory: /loki/chunks
```
What this config does:

* auth_enabled: false -- single-tenant mode, no authentication needed
* store: tsdb -- uses Loki's time-series database for indexing
* object_store: filesystem -- stores log chunks on local disk
* replication_factor: 1 -- single instance, no replication (fine for learning)
---

## 🐳 Docker Service (docker-compose.yml)

```yaml
loki:
  image: grafana/loki:latest
  container_name: loki
  ports:
    - "3100:3100"
  volumes:
    - ./loki/loki-config.yml:/etc/loki/loki-config.yml
    - loki_data:/loki
  command: -config.file=/etc/loki/loki-config.yml
  restart: unless-stopped
```
Add loki_data to your volumes section:

```yaml
volumes:
  prometheus_data:
  grafana_data:
  loki_data:
```
Start Loki:

```bash
docker compose up -d loki
```
---

## 🔍 Verify

```bash
curl http://localhost:3100/ready
```

Expected:

```
ready
```

---

# 📦 3. Promtail Setup

Promtail is the log collection agent. It reads Docker container log files from the host and pushes them to Loki.

```bash
mkdir -p promtail
```
## 📄 promtail/promtail-config.yml

```yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*-json.log
    pipeline_stages:
      - docker: {}
```
What this config does:

* positions -- tracks which log lines have already been shipped (like a bookmark)
* clients -- where to send logs (Loki endpoint)
* __path__ -- the glob pattern to find Docker JSON log files on the host
* pipeline_stages: docker: {} -- parses the Docker JSON log format and extracts timestamp, stream (stdout/stderr), and the log message
---

## 🐳 Docker Service (docker-compose.yml)

```yaml
promtail:
  image: grafana/promtail:latest
  container_name: promtail
  volumes:
    - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
    - /var/lib/docker/containers:/var/lib/docker/containers:ro
    - /var/run/docker.sock:/var/run/docker.sock
  command: -config.file=/etc/promtail/promtail-config.yml
  restart: unless-stopped
```
Why these volume mounts?

* /var/lib/docker/containers -- where Docker stores container log files (read-only)
* /var/run/docker.sock -- lets Promtail discover container metadata (names, labels)

Restart the stack:

```bash
docker compose up -d
```
---

## 🔍 Generate Logs

```bash
for i in $(seq 1 20); do curl -s http://localhost:8000 > /dev/null; done
```

---

# 🔌 4. Add Loki to Grafana

## 📄 grafana/provisioning/datasources/datasources.yml

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
```

---

## 🔄 Restart Grafana

```bash
docker compose restart grafana
```

---

# 🔎 5. LogQL Queries

## 🔹 All logs

```logql
{job="docker"}
```

---

## 🔹 Filter by container

```logql
{job="docker", container_name="notes-app"}
```

---

## 🔹 Search errors

```logql
{job="docker"} |= "error"
```

---

## 🔹 Regex (HTTP errors)

```logql
{job="docker"} |~ "status=[45]\\d{2}"
```

---

## 🔹 Count logs over time

```logql
count_over_time({job="docker"}[5m])
```

---

## 🔹 Rate of logs

```logql
rate({job="docker"}[5m])
```

---

## 🔹 Top containers by logs

```logql
topk(5, sum by (container_name) (rate({job="docker"}[5m])))
```

---

# 🎯 Challenge Answers

### 🔹 Error logs from notes-app (1 hour)

```logql
{job="docker", container_name="notes-app"} |= "error"
```

---

### 🔹 Error count per minute

```logql
rate({job="docker", container_name="notes-app"} |= "error" [1m])
```

---

# 🔗 6. Metrics + Logs Correlation

## Add Logs Panel

* Datasource: Loki
* Query:

```logql
{job="docker"}
```

---

## Explore Split View

### Left (Metrics)

```promql
rate(container_cpu_usage_seconds_total{name="notes-app"}[5m])
```

### Right (Logs)

```logql
{container_name="notes-app"}
```

---

## 🧠 Why this matters

Without correlation:

* Metrics → “CPU spike”
* Logs → “Error somewhere”

With correlation:

* See **exact log at exact spike time**

---

# ⚠️ Common Issues

### ❌ No logs in Grafana

Check:

```bash
docker logs promtail
```

---

### ❌ No files found

Check:

```
/var/lib/docker/containers/
```

---

### ❌ Wrong labels

Try:

```logql
{job="docker"}
```

Then inspect labels

---

# 🧠 Key Takeaways

* Loki = **efficient log storage**
* Promtail = **log shipper**
* LogQL = **query language for logs**
* Labels are **critical for filtering**
* Observability = **metrics + logs together**

---

# 🔥 What's Next?

Next step:

* Distributed tracing with **OpenTelemetry**
* Full 3-pillar observability stack

---

# 📌 Author

**Shibnath**
DevOps | Cloud | Observability Engineer 🚀

---
