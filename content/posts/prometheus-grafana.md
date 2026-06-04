---
title: "Monitoring with Prometheus and Grafana"
date: 2026-06-08
draft: false
tags: ["homelab", "monitoring", "prometheus", "grafana", "proxmox", "observability"]
series: ["Homelab Journey"]
description: "Building a full observability stack across all homelab containers — node_exporter on every host, Prometheus scraping every 15 seconds, and Grafana dashboards with Keycloak SSO."
cover:
  image: "/images/banners/banner-post-17.svg"
  alt: "Monitoring with Prometheus and Grafana"
  relative: false
---

Before adding monitoring, the only way to know if a container was struggling was to SSH in and run `htop`. Disk fills up silently. Memory pressure builds until something crashes. Prometheus and Grafana fix that — every host reports metrics every 15 seconds, Grafana visualises it, and eventually alerting will notify before things break rather than after.

This post covers the full stack: the `monitor` container setup, installing Prometheus and Grafana, deploying `node_exporter` across all midnight hosts, and configuring Grafana with Keycloak SSO so there's one login for everything.

---

## Architecture

```
leviathan, helm, bastion, gitea, keycloak, ipa, vault
        ↓ node_exporter (port 9100)
Prometheus (monitor · 192.168.40.40 · scrapes every 15s)
        ↓
Grafana (queries Prometheus via localhost)
        ↓
https://monitor.jordanaperry.com
```

All monitored hosts run `node_exporter` — a lightweight binary that exposes system metrics over HTTP on port 9100. Prometheus scrapes each target on a 15-second interval and stores the time-series data locally. Grafana queries Prometheus and renders dashboards.

---

## The container

CT 105 on `leviathan`, Debian 12, VLAN 40 (midnight):

```
Hostname: monitor
IP: 192.168.40.40/24
Gateway: 192.168.40.1
Storage: wreck-lvm (iSCSI)
```

One Proxmox-specific requirement: enable **Nesting** in the container features. Without it, some Grafana processes fail to start. In Proxmox: right-click CT 105 → **Options → Features → Nesting ✅**. If you're provisioning via Terraform, add `features { nesting = true }` to the container resource.

![Proxmox CT 105 monitor — Options → Features showing Nesting enabled](/images/TODO.png)
*Proxmox CT 105 — Nesting enabled under container features*

---

## Installing Prometheus

Prometheus isn't in the Debian repos at a current version. Install from the GitHub releases binary:

```bash
wget https://github.com/prometheus/prometheus/releases/download/v3.4.0/prometheus-3.4.0.linux-amd64.tar.gz
tar xvf prometheus-3.4.0.linux-amd64.tar.gz
mv prometheus-3.4.0.linux-amd64/prometheus /usr/local/bin/
mv prometheus-3.4.0.linux-amd64/promtool /usr/local/bin/
mkdir -p /etc/prometheus /var/lib/prometheus
```

Config at `/etc/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets:
          - '192.168.40.40:9100'   # monitor
          - '192.168.40.10:9100'   # leviathan
          - '192.168.40.20:9100'   # helm
          - '192.168.40.50:9100'   # bastion
          - '192.168.40.33:9100'   # gitea
          - '192.168.40.30:9100'   # keycloak
          - '192.168.40.31:9100'   # ipa
          - '192.168.40.32:9100'   # vault
```

The blog container (CT 101) is excluded — it's on twilight (VLAN 30) and the firewall rules correctly block midnight from reaching DMZ hosts.

Create a systemd service at `/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
useradd --no-create-home --shell /bin/false prometheus
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
systemctl enable --now prometheus
```

---

## Installing Grafana

Grafana is available via their apt repository:

```bash
apt install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | tee /etc/apt/sources.list.d/grafana.list
apt update && apt install grafana -y
systemctl enable --now grafana-server
```

---

## Connecting Grafana to Prometheus

In Grafana: **Connections → Data Sources → Add → Prometheus**

The URL must use the IP address, not localhost — Grafana runs as a separate process and the data source URL needs to be explicit:

```
URL: http://192.168.40.40:9090
```

Save and test. You should see "Successfully queried the Prometheus API."

![Grafana data source page showing Prometheus connected successfully](/images/TODO.png)
*Grafana → Data Sources — Prometheus connected at 192.168.40.40:9090*

---

## Importing the Node Exporter dashboard

Rather than building dashboards from scratch, import the community-maintained Node Exporter Full dashboard:

**Dashboards → Import → Dashboard ID: `1860`**

This gives you CPU usage, memory, disk I/O, network throughput, and load average for every scrape target. Use the **Nodename** dropdown in the dashboard to switch between hosts.

![Grafana Node Exporter Full dashboard showing CPU and memory across all hosts](/images/TODO.png)
*Node Exporter Full dashboard — all midnight containers visible in the dropdown*

---

## Installing node_exporter on all hosts

`node_exporter` needs to run on every host you want to monitor. The install is the same on each:

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
tar xvf node_exporter-1.9.1.linux-amd64.tar.gz
mv node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/
```

Systemd service at `/etc/systemd/system/node_exporter.service`:

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
useradd --no-create-home --shell /bin/false node_exporter
systemctl enable --now node_exporter
```

Repeat on: monitor, leviathan, helm, bastion, gitea, keycloak, ipa, vault.

Verify each target is reachable from the monitor container before adding it to `prometheus.yml`:

```bash
curl http://192.168.40.10:9100/metrics | head -5
```

---

## Keycloak SSO for Grafana

Once Keycloak is running (covered in a future post), Grafana can use it as an OIDC identity provider so the same lab credentials log into Grafana.

Key settings in `/etc/grafana/grafana.ini`:

```ini
[server]
domain = monitor.jordanaperry.com
root_url = https://monitor.jordanaperry.com

[auth.generic_oauth]
enabled = true
name = Keycloak
client_id = grafana
client_secret = your-client-secret
auth_url = https://keycloak.jordanaperry.com:8443/realms/homelab/protocol/openid-connect/auth
token_url = https://keycloak.jordanaperry.com:8443/realms/homelab/protocol/openid-connect/token
api_url = https://keycloak.jordanaperry.com:8443/realms/homelab/protocol/openid-connect/userinfo
skip_org_role_sync = false
allow_assign_grafana_admin = true
role_attribute_path = contains(roles[*], 'admin') && 'GrafanaAdmin' || 'Viewer'
```

One important detail: the role mapping uses `GrafanaAdmin` — not `Admin`. Using `Admin` doesn't work and Grafana just silently assigns the Viewer role instead.

---

## Current scrape targets

| Host | IP | Status |
|---|---|---|
| monitor | 192.168.40.40:9100 | ✅ UP |
| leviathan | 192.168.40.10:9100 | ✅ UP |
| helm | 192.168.40.20:9100 | ✅ UP |
| bastion | 192.168.40.50:9100 | ✅ UP |
| gitea | 192.168.40.33:9100 | ✅ UP |
| keycloak | 192.168.40.30:9100 | ✅ UP |
| ipa | 192.168.40.31:9100 | ✅ UP |
| vault | 192.168.40.32:9100 | ✅ UP |
| blog | excluded (DMZ firewall) | ⛔ |

![Prometheus targets page showing all hosts UP](/images/TODO.png)
*Prometheus → Targets — all midnight hosts reporting UP*

---

## What's next

Observability is in place — every midnight container is reporting metrics, dashboards are live, and Keycloak SSO handles authentication. Next up: Terraform — how all of this infrastructure is managed as code, why that matters for IAM roles, and how containers go from `terraform apply` to running in about 30 seconds.