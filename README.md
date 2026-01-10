# 📊 Central Monitoring Stack  
### **Prometheus + Alertmanager + Grafana with Node Exporter**

This repository runs a **single, central monitoring server** using **Docker Compose** to monitor system metrics and alerts across multiple servers.

---

## 🚀 What’s Included

The central monitoring server runs:

- 📈 **Prometheus** – Scrapes metrics & evaluates alert rules  
- 🚨 **Alertmanager** – Routes alerts (email, Slack, etc.)  
- 📊 **Grafana** – Dashboards & visualization  
- 🖥️ **Node Exporter** – Host-level metrics (CPU, RAM, Disk, Network)

It also supports scraping **additional servers** (Server-2, Server-3, …), each running **Node Exporter**.

> ✅ **Production Best Practice**
>
> - Keep Prometheus `instance` label as **IP:PORT**  
> - Add extra labels like `hostname` and `ip`  
> - Many Grafana dashboards (including Node Exporter Full) depend on this

---

## 📁 Folder Structure

```text
└── monitoring
    ├── README.md
    ├── docker-compose.yml
    │
    ├── alertmanager
    │   └── alertmanager.yml
    │
    ├── grafana
    │   └── provisioning
    │       ├── dashboards
    │       │   └── dashboard.yml
    │       └── datasources
    │           └── datasource.yml
    │
    └── prometheus
        ├── prometheus.yml
        └── rules
            └── alerts.yml
```

---

## 🔌 Service Ports

| Service         | Port | URL |
|-----------------|-----:|-----|
| Prometheus      | 9090 | http://<MONITORING_SERVER_IP>:9090 |
| Alertmanager    | 9093 | http://<MONITORING_SERVER_IP>:9093 |
| Grafana         | 3000 | http://<MONITORING_SERVER_IP>:3000 |
| Node Exporter   | 9100 | http://<NODE_IP>:9100/metrics |

---

## 🧰 Prerequisites

On the **monitoring server**:

- Docker Engine
- Docker Compose Plugin

```bash
curl -fsSL https://get.docker.com | bash
sudo usermod -aG docker $USER
newgrp docker

docker images
docker ps
```

---

## ▶️ 1) Start the Monitoring Stack

From the `monitoring/` directory:

```bash
docker compose up -d
docker compose ps
```

### ✅ Verify Prometheus Targets

Open:

```
http://<MONITORING_SERVER_IP>:9090/targets
```

You should see:

- `prometheus` → **UP**
- `node` → **UP** (node-exporter:9100)

---

## 📊 2) Grafana – Node Exporter Dashboard

Login:

- URL: http://<MONITORING_SERVER_IP>:3000  
- Default credentials:
  ```
  admin / admin123
  ```

### Import Dashboard

- 📌 **Dashboard ID**: **1860**
- Name: **Node Exporter Full**

Then select your server using the **Instance** dropdown.

---

## ➕ 3) Add More Servers (Server-2, Server-3, …)

### A) Install Node Exporter (Docker)

On **each additional server**:

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

Start Node Exporter:

```bash
docker compose up -d
docker compose ps
```

Verify metrics:

```bash
curl http://localhost:9100/metrics | head
```

---

### 🔐 B) Firewall Recommendation

Allow **port 9100 only from the monitoring server**.

Reload Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload
```

---

## 🚨 4) Alerts (CPU, RAM, Disk)

Alerts are evaluated **centrally** and apply to **all servers**.

Edit:

```
monitoring/prometheus/rules/alerts.yml
```

Included alerts:

- ⚠️ High CPU usage  
- ⚠️ High RAM usage  
- ⚠️ Low disk space on `/`

Reload Prometheus after changes:

```bash
curl -X POST http://localhost:9090/-/reload
```

---

## ✉️ 5) Email Notifications (Alertmanager)

Edit:

```
monitoring/alertmanager/alertmanager.yml
```

⚠️ **Security Warning**

❌ Do NOT commit real SMTP passwords to GitHub  
✅ Use:
- `.env` + environment variables  
- Docker secrets  
- CI/CD secret store  

Restart Alertmanager:

```bash
docker compose restart alertmanager
```

---

## 🧪 6) Test RAM Alert

```bash
stress --vm 1 --vm-bytes 256M --vm-keep --timeout 5m
pkill stress
```

---

## 🛠️ Troubleshooting

### Node Exporter is UP but Grafana shows N/A
- Do not overwrite `instance` label
- Dashboard 1860 expects `instance=IP:PORT`

### Prometheus cannot scrape server
```bash
curl http://<SERVER_IP>:9100/metrics | head
```

### Reload endpoint not working
Ensure Prometheus runs with:
```
--web.enable-lifecycle
```

---

## 🔐 Security Best Practices

- Do NOT expose port **9100** publicly
- Restrict access via VPN / trusted IPs
- Store SMTP credentials securely

---

## 📜 License

MIT
