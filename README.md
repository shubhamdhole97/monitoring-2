# Central Monitoring Stack (Prometheus + Alertmanager + Grafana) with Node Exporter

This repository runs a **single, central monitoring server** in Docker:
- **Prometheus** (scrapes metrics + evaluates alert rules)
- **Alertmanager** (routes alerts to email, etc.)
- **Grafana** (dashboards)
- **Node Exporter** (host metrics for the monitoring server itself)

It also supports scraping **additional servers** (Server-2, Server-3, …) that run **Node Exporter**.

> ✅ Production best practice used here: keep Prometheus `instance` label as `IP:PORT` (Grafana dashboards depend on it), and add `hostname` + `ip` labels for human-friendly alert messages.

---

## Folder structure

```text
└── monitoring
    ├── README.md
    ├── alertmanager
    │   └── alertmanager.yml
    ├── docker-compose.yml
    ├── grafana
    │   └── provisioning
    │       ├── dashboards
    │       │   └── dashboard.yml
    │       └── datasources
    │           └── datasource.yml
    └── prometheus
        ├── prometheus.yml
        └── rules
            └── alerts.yml


```

---

## Ports

| Service | Port | URL |
|---|---:|---|
| Prometheus | 9090 | `http://<MONITORING_SERVER_IP>:9090` |
| Alertmanager | 9093 | `http://<MONITORING_SERVER_IP>:9093` |
| Grafana | 3000 | `http://<MONITORING_SERVER_IP>:3000` |
| Node Exporter | 9100 | `http://<NODE_IP>:9100/metrics` |

---

## Prerequisites

On the monitoring server:
-  Install Docker Engine
- Docker Compose plugin (`docker compose`)

```bash
curl -fsSL https://get.docker.com | bash
sudo usermod -aG docker $USER
newgrp docker
docker images
docker ps
```

---

## 1) Start the monitoring stack

From the `monitoring/` directory:

```bash
docker compose up -d
docker compose ps
```

### Verify targets in Prometheus
Open:
- `http://<MONITORING_SERVER_IP>:9090/targets`

You should see:
- `prometheus` → **UP**
- `node` → **UP** (at least `node-exporter:9100`)

---

## 2) Grafana (Node Exporter Full dashboard)

Login:
- `http://<MONITORING_SERVER_IP>:3000`
- Default user/pass (if you didn't change): `admin / admin123`

Import the popular dashboard:
- Dashboard ID: **1860** (Node Exporter Full)

Then choose your node using the **Instance** dropdown.

---

## 3) Add more servers (Server-2, Server-3, …)

### A) Install & run Node Exporter on each new server (Docker Compose)

On **Server-2 / Server-3**:

```bash
mkdir -p ~/node-exporter
cd ~/node-exporter
```

Create `docker-compose.yml`:

```yaml
services:
  node-exporter:
    image: prom/node-exporter:v1.8.2
    container_name: node-exporter
    restart: unless-stopped
    network_mode: host
    pid: host
    command:
      - '--path.rootfs=/host'
    volumes:
      - '/:/host:ro,rslave'
```

Start:

```bash
docker compose up -d
docker compose ps
```

Verify:

```bash
curl http://localhost:9100/metrics | head
```

### B) Allow port 9100 only from monitoring server (recommended)






Reload Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload
```

---

## 4) Alerts (CPU, RAM, Disk) for all servers

Alerts are evaluated centrally by Prometheus and apply to **every** scraped node (Server-1, Server-2, Server-3…).

Edit: `monitoring/prometheus/rules/alerts.yml`

This setup includes:
- High CPU usage
- High RAM usage
- Low disk space on `/`

Reload after changes:

```bash
curl -X POST http://localhost:9090/-/reload
```

### Where to view alerts
- Prometheus alerts: `http://<MONITORING_SERVER_IP>:9090/alerts`
- Alertmanager: `http://<MONITORING_SERVER_IP>:9093`

---

## 5) Email notifications (Alertmanager)

Edit: `monitoring/alertmanager/alertmanager.yml`

⚠️ **Do not commit real passwords** to GitHub. Use:
- `.env` + environment variables, or
- Docker secrets, or
- your CI/CD secrets store

After changes, restart Alertmanager:

```bash
docker compose restart alertmanager
```

---

## 6) Test RAM alert (simple)

On a target server (Server-2/Server-3), run:

```bash
stress --vm 1 --vm-bytes 512M --vm-keep --timeout 5m
```

If you see `Killed (signal 9)`, reduce the memory size:

```bash
stress --vm 1 --vm-bytes 256M --vm-keep --timeout 5m
```

Stop early:

```bash
pkill stress
```

---

## Troubleshooting

### Node exporter is UP in Prometheus but Grafana shows N/A
- Ensure you **did not overwrite** the Prometheus `instance` label.
- Node Exporter Full (1860) expects `instance=IP:PORT` in many queries.

### Prometheus cannot scrape server
On monitoring server:

```bash
curl http://<SERVER_IP>:9100/metrics | head
```

If it fails:
- open firewall/security group for 9100 (from monitoring server only)
- confirm node-exporter container is running

### Reload not working
If `/-/reload` fails, ensure Prometheus container runs with `--web.enable-lifecycle` in `docker-compose.yml`.

---

## Security notes (recommended)

- Do **not** expose 9100 publicly.
- Restrict Grafana/Prometheus/Alertmanager to VPN / trusted IP ranges.
- Store SMTP credentials securely (env vars / secrets).

---

## License

MIT (or update as you prefer).
