# 📊 Day 02 – Node Exporter, cAdvisor & Grafana Dashboards

Welcome to **Day 02** of the *DevOps Observability Stack*.

In this module, we extend our monitoring system beyond Prometheus itself and start collecting **real infrastructure and container-level metrics**, then visualize them using **Grafana dashboards**.

---

# 🚀 What You Will Learn

* Host monitoring using **Node Exporter**
* Container monitoring using **cAdvisor**
* Visualization using **Grafana**
* Building **custom dashboards**
* Automating Grafana setup using **provisioning (YAML)**

---

# 🏗️ Architecture Overview

```
[Host] ------> Node Exporter ----\
                                  \
[Docker] ---> cAdvisor -----------> Prometheus ---> Grafana Dashboards
                                  /
[Prometheus] --------------------/
```

---

# 📁 Project Structure

```
devops-observability-stack/
└── day-02/
    ├── docker-compose.yml
    ├── prometheus.yml
    └── grafana/
        └── provisioning/
            ├── datasources/
            │   └── datasources.yml
            └── dashboards/
```

---

# ⚙️ 1. Node Exporter (Host Metrics)

## 📌 What it does

Exposes:

* CPU usage
* Memory usage
* Disk usage
* Network statistics

---

## 🐳 Service Configuration (docker-compose.yml)

```yaml
node-exporter:
  image: prom/node-exporter:latest
  container_name: node-exporter
  ports:
    - "9100:9100"
  volumes:
    - /proc:/host/proc:ro
    - /sys:/host/sys:ro
    - /:/rootfs:ro
  command:
    - '--path.procfs=/host/proc'
    - '--path.sysfs=/host/sys'
    - '--path.rootfs=/rootfs'
    - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
  restart: unless-stopped
```
Why these volume mounts?

* /proc -- kernel and process information (CPU stats, memory info)
* /sys -- hardware and driver details
* -- filesystem usage (disk space)
All mounted read-only (ro) -- Node Exporter only reads, never modifies.

Add it as a scrape target in prometheus.yml:
```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```
Restart the stack:

```bash
docker compose up -d
```
---

## 🔍 Verify

```bash
curl http://localhost:9100/metrics | head -20
```

---

## 📊 Sample Queries

```promql
# CPU: percentage of time spent idle (per core)
node_cpu_seconds_total{mode="idle"}

# Memory: total vs available
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes

# Memory usage percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk: filesystem usage percentage
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100

# Network: bytes received per second
rate(node_network_receive_bytes_total[5m])
```

---

# 🐳 2. cAdvisor (Container Metrics)

## 📌 What it does

Tracks:

* Container CPU usage
* Memory usage
* Network usage
* Filesystem usage

---

## 🐳 Service Configuration (Add it to your docker-compose.yml)

```yaml
cadvisor:
  image: gcr.io/cadvisor/cadvisor:latest
  container_name: cadvisor
  ports:
    - "8080:8080"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
  restart: unless-stopped
```
Why these volume mounts?

* Docker socket (docker.sock) -- lets cAdvisor discover and query running containers
* /sys -- kernel-level container stats (cgroups)
* /var/lib/docker/ -- container filesystem information
Add cAdvisor as a Prometheus scrape target:

```yaml
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]
```
Restart the stack:

```bash
docker compose up -d
```
---

## 🔍 Verify

Open:

```
http://localhost:8080
```

---

## 📊 Sample Queries

```promql
# CPU usage per container (in seconds)
rate(container_cpu_usage_seconds_total{name!=""}[5m])

# Memory usage per container
container_memory_usage_bytes{name!=""}

# Network received bytes per container
rate(container_network_receive_bytes_total{name!=""}[5m])

# Which container is using the most memory?
topk(3, container_memory_usage_bytes{name!=""})
```

---

# ⚖️ Node Exporter vs cAdvisor

| Feature  | Node Exporter              | cAdvisor                     |
| -------- | -------------------------- | ---------------------------- |
| Scope    | Host-level                 | Container-level              |
| Metrics  | CPU, Memory, Disk, Network | Container CPU, Memory, FS    |
| Use Case | VM/Server monitoring       | Docker/Kubernetes monitoring |

👉 Use both together for **complete visibility**

---

# 🔧 3. Prometheus Configuration

Update `prometheus.yml`:

```yaml
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
```

---

# 📊 4. Grafana Setup

Grafana is the visualization layer. It connects to Prometheus (and later Loki) and lets you build dashboards, set alerts, and share views with your team.

## 🐳 Service Configuration (Add Grafana to your docker-compose.yml)

```yaml
grafana:
  image: grafana/grafana-enterprise:latest
  container_name: grafana
  ports:
    - "3000:3000"
  volumes:
    - grafana_data:/var/lib/grafana
    - ./grafana/provisioning:/etc/grafana/provisioning
  environment:
    - GF_SECURITY_ADMIN_USER=admin
    - GF_SECURITY_ADMIN_PASSWORD=admin123
  restart: unless-stopped
```
Add the volume at the bottom of your compose file:

```yaml
volumes:
  prometheus_data:
  grafana_data:
```
Restart:
```bash
docker compose up -d
```
---

## 🔐 Login

```
http://localhost:3000
```

Username: `admin`
Password: `admin123`


Add Prometheus as a datasource:

1. Go to Connections > Data Sources > Add data source
2. Select Prometheus
3. Set URL to http://prometheus:9090 (use the container name, not localhost -- they are on the same Docker network)
4. Click Save & Test -- you should see "Successfully queried the Prometheus API"

---

# ⚙️ 5. Auto-Provision Datasource

In production, you do not click through the UI to add datasources. You provision them with configuration files so the setup is repeatable.

Create the provisioning directory structure:
```bash
mkdir -p grafana/provisioning/datasources
mkdir -p grafana/provisioning/dashboards
```
## 📄 grafana/provisioning/datasources/datasources.yml:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```
Restart Grafana:
```bash
docker compose up -d grafana
```
Check Connections > Data Sources -- Prometheus should already be there without any manual setup.
---

## 💡 Why provisioning?

* Repeatable setup
* No manual UI steps
* Infrastructure as Code approach
* Useful in CI/CD pipelines

---

# 📊 6. Custom Dashboard Panels

### 🔹 CPU Usage %

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

---

### 🔹 Memory Usage %

```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

---

### 🔹 Container CPU Usage

```promql
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
```

---

### 🔹 Container Memory (MB)

```promql
container_memory_usage_bytes{name!=""} / 1024 / 1024
```

---

### 🔹 Disk Usage %

```promql
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
```

---

# 📥 7. Import Community Dashboards

## Node Exporter Dashboard

* ID: **1860**

## cAdvisor Dashboard

* ID: **193**

---

# ▶️ 8. Run the Stack

```bash
docker compose up -d
```

---

## ✅ Verify

```bash
docker compose ps
```

Ensure:

* prometheus → UP
* node-exporter → UP
* cadvisor → UP
* grafana → UP

---

# 🧠 Key Takeaways

* Node Exporter → **host metrics**
* cAdvisor → **container metrics**
* Grafana → **visualization layer**
* Provisioning → **production-grade automation**

---

# 🔥 What's Next?

Next steps in the observability stack:

* Log aggregation with **Loki**
* Log collection with **Promtail**
* Distributed tracing with **OpenTelemetry**

---

# 📌 Author

**Shibnath**
DevOps | Cloud | Observability Engineer 🚀

---
