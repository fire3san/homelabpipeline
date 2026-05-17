# 📊 Prometheus, Grafana & PVE Exporter Setup Guide

> Deploy a LAN-only monitoring stack on `docker-ubuntu` that collects metrics from Proxmox, Traefik, and Prometheus itself — and visualises them in Grafana dashboards.

All three containers sit on `proxy-network` with `traefik.enable=false` so they are **never exposed to the internet**.

---

## 📋 Prerequisites

- Completed [DockerInstallSetupGuide.md](DockerInstallSetupGuide.md) — `proxy-network` must exist.
- SSH access to `docker-ubuntu` (192.168.1.12) as `ubuntu`.
- A running Proxmox VE host at 192.168.1.10 (or whatever your PVE IP is).
- (Recommended) Traefik already deployed — the stack scrapes Traefik's metrics endpoint at `:8080`.

---

## 🪜 Step 1 — Create a Proxmox API token

PVE Exporter reads the Proxmox API to collect VM/CPU/RAM metrics. You need an API token for it.

1. Open the Proxmox web UI at `https://192.168.1.10:8006`.
2. Go to **Datacenter → Permissions → API Tokens**.
3. Click **Add**.
   - User: select or create a user (e.g. `root@pam` or a dedicated `prometheus@pve` user).
   - Token ID: `monitoring`.
   - **Uncheck "Privilege Separation"** for simplicity — this gives the token the same rights as the user. If you prefer least-privilege, leave it checked but you must also add a Token Permission row below.
4. Copy the **secret** value that appears — it is shown only once.

> ⚠️ If you leave **Privilege Separation** checked, you must also go to **Datacenter → Permissions → Add → Token Permission**, select the `user@pve!monitoring` path, and grant at least `PVEAuditor` on `/`. Without this, every scrape will return `403 Forbidden`.

> 📸 *Screenshot placeholder: Proxmox API Token creation dialog with "Privilege Separation" unchecked.*

---

## 🪜 Step 2 — Prepare the config files on the host

SSH into `docker-ubuntu`:

```bash
ssh ubuntu@192.168.1.12
```

Create the required directories:

```bash
mkdir -p /home/ubuntu/prometheus/data
mkdir -p /home/ubuntu/grafana/data
mkdir -p /home/ubuntu/prometheus/config
```

### Copy the Prometheus config

```bash
# If you already cloned this repo:
cp ~/homelabpipeline/stacks/monitoring/config/prometheus.yml /home/ubuntu/prometheus/prometheus.yml

# Otherwise create it manually (see stacks/monitoring/config/prometheus.yml for the full content)
```

### Create the PVE Exporter config

```bash
nano /home/ubuntu/prometheus/pve.yml
```

Paste the following, replacing the placeholder values with the token you created in Step 1:

```yaml
default:
  user: prometheus@pve
  password:
  verify_ssl: false

# If using an API token (recommended):
# default:
#   user: prometheus@pve!monitoring
#   token: <paste-your-api-token-secret-here>
#   verify_ssl: false
```

> 🪪 The `pve.yml` file contains credentials. Add it to `.gitignore` if you version this directory.

> ⚠️ Prometheus strictly rejects **Tab characters** in its config. If you edit `prometheus.yml` with `nano`, enable `tabstospaces`:
> ```bash
> echo "set tabstospaces" >> ~/.nanorc
> ```

> 📸 *Screenshot placeholder: `cat /home/ubuntu/prometheus/prometheus.yml` showing the three scrape targets.*

---

## 🪜 Step 3 — Deploy the monitoring stack

Copy the compose file from this repo:

```bash
# If you already cloned this repo:
cp ~/homelabpipeline/stacks/monitoring/docker-compose.yml /home/ubuntu/monitoring-compose.yml

# Otherwise create it manually — see stacks/monitoring/docker-compose.yml for the full content
```

Or deploy via **Portainer**:

1. Open Portainer at `https://192.168.1.12:9443`.
2. Go to **Stacks → Add stack**.
3. Name: `monitoring`.
4. Paste the contents of `stacks/monitoring/docker-compose.yml`.
5. Click **Deploy the stack**.

Or deploy from the CLI:

```bash
cd ~/homelabpipeline/stacks/monitoring
docker compose up -d
```

Wait a few seconds and check that all three containers are running:

```bash
docker ps --filter "name=prometheus" --filter "name=grafana" --filter "name=pve-exporter"
```

Expected: three containers, all `Up`.

> 📸 *Screenshot placeholder: `docker ps` showing prometheus, grafana, and pve-exporter all running.*

---

## 🪜 Step 4 — Verify Prometheus targets

Open Prometheus targets in your browser:

```
http://192.168.1.12:9090/targets
```

You should see three targets, all with state **UP**:

| Job | Target | Expected state |
|-----|--------|---------------|
| `prometheus` | `localhost:9090` | UP |
| `pve-exporter` | `pve-exporter:9221` | UP |
| `traefik` | `traefik:8080` | UP |

> If `pve-exporter` is **DOWN**, check that `pve.yml` has valid credentials and that the Proxmox API is reachable from the Docker network.

> 📸 *Screenshot placeholder: Prometheus /targets page showing all three targets UP with green labels.*

---

## 🪜 Step 5 — Configure Grafana

Open Grafana in your browser:

```
http://192.168.1.12:3030
```

First login:
- Username: `admin`
- Password: `admin`
- Grafana will prompt you to set a new password — choose a strong one and save it.

### Add the Prometheus data source

1. Go to **Configuration → Data Sources → Add data source**.
2. Type: **Prometheus**.
3. URL: `http://prometheus:9090` (use the Docker service name, **not** `localhost`).
4. Click **Save & Test**. You should see a green "Data source is working" banner.

> ⚠️ The URL must be `http://prometheus:9090` — the Docker DNS resolves this to the Prometheus container. Using `localhost:9090` will fail because Grafana's localhost is the Grafana container itself.

### Import a Proxmox dashboard

1. Go to **Dashboards → Import**.
2. Enter dashboard ID **10347** (the popular "Proxmox via Prometheus" dashboard by prompve).
3. Click **Load**.
4. Select the Prometheus data source you just created.
5. Click **Import**.

You should now see a dashboard with CPU, RAM, disk, and VM status metrics from your Proxmox host.

### Import a Traefik dashboard (optional)

1. Go to **Dashboards → Import**.
2. Enter dashboard ID **11462** (Traefik official dashboard) or search for "Traefik" in the Grafana dashboard library.
3. Click **Load**, select your Prometheus data source, and click **Import**.

> 📸 *Screenshot placeholder: Grafana dashboard showing Proxmox VM metrics.*

---

## 🪜 Step 6 — (Optional) Persistent Grafana dashboards

By default, Grafana stores dashboards in its SQLite database inside the container volume. If you ever recreate the container, any dashboards you created through the UI will be lost unless you:

- **Export dashboards as JSON** and re-import them after a rebuild, or
- **Provision dashboards via config files** by mounting a `_dashboards/` directory into `/etc/grafana/provisioning/dashboards/`.

For a homelab, exporting JSON after major changes and re-importing is usually good enough.

---

## 🪜 Step 7 — Snapshot

Back in the Proxmox UI: `docker-ubuntu` → **Snapshots** → **Take Snapshot** → name `monitoring-installed`.

---

## 🔌 Ports opened in this guide

| Port | Proto | Where | Reachable from | Notes |
|------|-------|-------|----------------|-------|
| `8006` | TCP | PVE Exporter container → Proxmox host (`192.168.1.10`) | LAN only | Proxmox VE API. PVE Exporter scrapes metrics from this endpoint. Must be reachable from the `proxy-network` Docker bridge. |
| `9090` | TCP | Prometheus container (host port `9090`) | LAN only | Prometheus Web UI and query API. |
| `9221` | TCP | PVE Exporter container | Docker internal only | Scraped by Prometheus over `proxy-network`. No host port mapping needed. |
| `3000` | TCP | Grafana container (internal) | — | Grafana's default listen port inside the container. |
| `3030` | TCP | `docker-ubuntu` host port (maps to container `3000`) | LAN only | Grafana dashboards in the browser. Default login: `admin` / `admin` (change on first visit). |

> 🛡️ All three monitoring services have `traefik.enable=false` and are **never exposed via the Cloudflare Tunnel**. Access them only from the LAN. Port `8006` is already open on the Proxmox host — the PVE Exporter container reaches it through the Docker host's network via `proxy-network`.

---

## ✅ Done — what's next?

You now have full observability over your Proxmox host, VMs, and Traefik routing. Continue with:

➡️ Back to the [README](README.md#-adding-a-new-app) to deploy your first app, or review the [monitoring section](README.md#-monitoring) for tips on dashboards and troubleshooting.

---

## 🛟 Troubleshooting

| Symptom | Fix |
|---------|-----|
| 📛 PVE Exporter target is **DOWN** in Prometheus | Check `pve.yml` credentials. If using an API token with Privilege Separation, add a Token Permission row in Proxmox for `user@pve!tokenid`. |
| 📛 `Error loading config: did not find expected key` | `prometheus.yml` contains **Tab characters**. Replace all tabs with spaces. Run `expand -t 4 prometheus.yml > prometheus-fixed.yml && mv prometheus-fixed.yml prometheus.yml` then restart. |
| 📊 Grafana shows **"no data"** on dashboards | Confirm the data source URL is `http://prometheus:9090` (Docker DNS, not `localhost`). Check that Prometheus targets are all UP at `http://192.168.1.12:9090/targets`. |
| 🚫 Port 3000 conflict — Grafana won't start | The compose file maps port `3030:3000` to avoid this. Access Grafana on port `3030`, not `3000`. |
| 🔁 PVE Exporter returns `403 Forbidden` | The Proxmox API token has Privilege Separation enabled without a Token Permission row. Either uncheck Privilege Separation when creating the token, or add a permission row at Datacenter → Permissions. |
| 🐳 Docker DNS failure: `lookup prometheus on 127.0.0.11:53` | A target container is in a crash loop. Check `docker ps -a` and `docker logs <container>` to find the crash cause. |
| 🐢 Metrics are delayed or stale | Default scrape interval is `15s`. If PVE Exporter is slow to respond, Proxmox may be under load. Check `http://192.168.1.12:9090/targets` for the last scrape duration. |