# Redis Monitoring Stack

A complete Redis monitoring setup using **redis_exporter**, **Prometheus**, and **Grafana** — deployable on Kubernetes via Helm.

This repository covers the full pipeline:
- Installing `redis_exporter` as systemd services on Redis hosts
- Scraping metrics with Prometheus using file-based service discovery
- Visualizing everything in a pre-built Grafana dashboard

---

## Repository Structure

```
redis-monitoring/
├── README.md
├── grafana/
│   ├── values.yaml                        # Grafana Helm values
│   └── dashboards/
│       └── redis-cluster-overview.json    # Pre-built Grafana dashboard
├── prometheus/
│   ├── values.yaml                        # Prometheus Helm values
│   └── configmap-redis-targets.yaml       # Redis exporter targets (file_sd)
└── redis-exporter/
    └── setup_redis_exporter.sh            # Exporter install script (runs on Redis hosts)
```

---

## Prerequisites

- Kubernetes cluster with `kubectl` access
- Helm 3
- A `monitoring` namespace: `kubectl create namespace monitoring`
- A StorageClass available in your cluster (update `storageClassName` in both `values.yaml` files)
- Redis hosts reachable from the Prometheus pod (network/firewall rules allowing scrape ports)

---

## Step 1 — Install redis_exporter on Redis Hosts

Run the setup script on **each Redis host**. It installs `redis_exporter` and creates systemd services for each Redis instance on the host.

```bash
sudo ./redis-exporter/setup_redis_exporter.sh --host <redis_host_ip_or_hostname>
```

### Script Options

| Flag | Description | Required |
|------|-------------|----------|
| `--host` | Redis host IP or hostname | Yes |
| `--bind` | Exporter bind address (default: `0.0.0.0`) | No |
| `--redis-user` | Redis ACL username | No |
| `--redis-pass` | Redis password | No |

### Examples

```bash
# Basic
sudo ./redis-exporter/setup_redis_exporter.sh --host localhost

# Bind to specific interface
sudo ./redis-exporter/setup_redis_exporter.sh --host 127.0.0.1 --bind 0.0.0.0

# With Redis authentication
sudo ./redis-exporter/setup_redis_exporter.sh --host localhost --redis-user monitoring --redis-pass 'secret'
```

### What the script does

The script automatically:

1. Downloads and installs `redis_exporter` to `/opt/redis_exporter/`
2. Creates three systemd services — one per Redis instance:

| Redis Port | Exporter Port | Systemd Service |
|------------|---------------|-----------------|
| 7001 | 9121 | `redis-exporter-7001.service` |
| 7002 | 9122 | `redis-exporter-7002.service` |
| 7003 | 9123 | `redis-exporter-7003.service` |

3. Enables and starts all three services

### Verify the installation

```bash
# Check service status
systemctl status redis-exporter-7001
systemctl status redis-exporter-7002
systemctl status redis-exporter-7003

# Confirm listening ports
ss -tunlp | egrep ':(9121|9122|9123)'

# Test metrics endpoints
curl http://127.0.0.1:9121/metrics
curl http://127.0.0.1:9122/metrics
curl http://127.0.0.1:9123/metrics

# Quick sanity check
curl -s http://127.0.0.1:9121/metrics | grep redis_connected_clients
```

---

## Step 2 — Configure Prometheus Targets

Edit `prometheus/configmap-redis-targets.yaml` to list your Redis exporter addresses grouped by cluster.

Each group represents one logical Redis cluster and must have a `redis_cluster` label — this is what the Grafana dashboard uses for filtering.

```json
[
  {
    "targets": [
      "redis-node-01.prod.example.local:9121",
      "redis-node-01.prod.example.local:9122",
      "redis-node-02.prod.example.local:9121"
    ],
    "labels": {
      "redis_cluster": "prod-redis-cache"
    }
  }
]
```

> **Note:** The `environment` label is derived automatically from the hostname via Prometheus `relabel_configs`. Adjust the regex in `prometheus/values.yaml` to match your hostname convention.

Apply the ConfigMap **before** deploying Prometheus:

```bash
kubectl apply -f prometheus/configmap-redis-targets.yaml
```

> After the initial deploy, you can add or remove targets by editing and re-applying this ConfigMap. Prometheus reloads file_sd targets automatically — no restart needed.

---

## Step 3 — Deploy Prometheus

Edit `prometheus/values.yaml`:
- Set `storageClass` to your cluster's StorageClass name

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  -f prometheus/values.yaml
```

---

## Step 4 — Deploy Grafana

Edit `grafana/values.yaml`:
- Set `storageClassName` to your cluster's StorageClass name
- Set `adminPassword` to a secure password
- Configure `smtp` section if you want alert email notifications

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install grafana grafana/grafana \
  --namespace monitoring \
  -f grafana/values.yaml
```

> The Prometheus datasource is provisioned automatically with UID `prometheus`. This UID is hardcoded in the dashboard JSON — do not change it in `values.yaml`.

---

## Step 5 — Import the Grafana Dashboard

1. Open Grafana in your browser
2. Go to **Dashboards → Import**
3. Upload `grafana/dashboards/redis-cluster-overview.json`
4. Click **Import**

---

## Dashboard Overview

The **Redis Cluster Overview** dashboard is organized into collapsible sections:

| Section | What it shows |
|---------|---------------|
| **Cluster Summary** | Total ops/sec, memory used, client count, cache hit ratio across the whole cluster |
| **Health & Clients** | Per-host UP/DOWN status, connected clients over time, blocked clients |
| **Performance** | Ops per second, keyspace hits vs misses, hit/miss stats for selected time range |
| **Command Operations** | GET count, total commands, GET ops/sec, command distribution by type |
| **Command Insights** | Top 10 commands by rate, top 10 slowest commands by avg latency, keyspace size by DB |
| **Slowlog** | Slowlog length per instance, last slow execution duration, slowlog ID trend |
| **Cache Efficiency** | Cache hit ratio over time, miss rate per second, GET % of total ops |
| **Cluster Balance** | Key distribution by node, key imbalance ratio (max/avg) |
| **CPU** | Redis CPU usage per instance (user + sys) |
| **Memory** | Memory used per instance, fragmentation ratio, evicted keys/sec, expired keys/sec |
| **Network & Persistence** | Network in/out bytes/sec, AOF enabled state, RDB snapshot age |
| **Troubleshooting Table** | Raw target status table, filterable by host/port |

### Dashboard Variables

| Variable | Description |
|----------|-------------|
| `cluster` | Filter by Redis cluster name (from `redis_cluster` label) |
| `env` | Filter by environment (derived from hostname) |
| `host` | Filter by individual Redis host |
| `xport` | Filter by exporter port |

All variables support multi-select and default to **All**.

---

## Key Metrics Reference

| Metric | Description |
|--------|-------------|
| `redis_up` | Instance reachability (1=UP, 0=DOWN) |
| `redis_connected_clients` | Active client connections |
| `redis_blocked_clients` | Clients blocked on BLPOP/BRPOP/etc |
| `redis_memory_used_bytes` | Current memory usage |
| `redis_mem_fragmentation_ratio` | Memory fragmentation ratio |
| `redis_keyspace_hits_total` | Cumulative cache hits |
| `redis_keyspace_misses_total` | Cumulative cache misses |
| `redis_commands_processed_total` | Total commands processed |
| `redis_evicted_keys_total` | Keys evicted due to memory limit |
| `redis_expired_keys_total` | Keys expired by TTL |
| `redis_slowlog_length` | Current slowlog entry count |

---

## Credits

- [redis_exporter](https://github.com/oliver006/redis_exporter) by Oliver006
- [Prometheus](https://prometheus.io/)
- [Grafana](https://grafana.com/)