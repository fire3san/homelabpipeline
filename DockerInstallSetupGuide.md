# 🐳 Docker Install Setup Guide

> Install Docker Engine + Compose plugin on `docker-ubuntu` and create the shared `proxy-network` bridge that ties Traefik to every app container.

This is a short guide, but **do not skip the `proxy-network` step** — it is the single most common cause of "Traefik can't see my container" 404s later.

---

## 📋 Prerequisites

- Completed [ProxmoxVMSetupGuide.md](ProxmoxVMSetupGuide.md).
- SSH access to `docker-ubuntu` (192.168.1.12) as `ubuntu`.

---

## 🪜 Step 1 — Install Docker Engine

The official one-liner is the cleanest path:

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Verify:

```bash
docker --version
docker compose version
```

> 📸 *Screenshot placeholder: terminal showing `Docker version 27.x` and `Docker Compose version v2.x`.*

---

## 🪜 Step 2 — Run Docker as non-root (optional but recommended)

```bash
sudo usermod -aG docker $USER
# Log out and back in so the group takes effect
exit
ssh ubuntu@192.168.1.12
docker run --rm hello-world
```

If `hello-world` prints its greeting, you're set.

> ⚠️ Anyone in the `docker` group is effectively **root on the host** (because they can mount `/` into a container). Don't add untrusted users.

---

## 🪜 Step 3 — Create the `proxy-network` bridge

This is the network that **Traefik and every app container will share**. Without it, Traefik literally cannot see your apps.

```bash
docker network create \
  --driver bridge \
  --subnet 172.18.0.0/16 \
  proxy-network
```

Verify:

```bash
docker network inspect proxy-network --format '{{.Name}} {{.IPAM.Config}}'
# Expected: proxy-network [{172.18.0.0/16  ...}]
```

> 💡 Pinning the subnet (`172.18.0.0/16`) is not strictly required, but it makes your `iptables`/firewall rules deterministic.

---

## 🪜 Step 4 — Configure log rotation

By default Docker logs grow forever. Cap them:

```bash
sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
sudo systemctl restart docker
```

---

## 🪜 Step 5 — (Optional) Enable the firewall

`ufw` is fine here. We're only allowing LAN traffic.

```bash
sudo apt -y install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp     # SSH
sudo ufw allow from 192.168.1.0/24 to any port 80 proto tcp     # Traefik ingress (cloudflared)
sudo ufw allow from 192.168.1.0/24 to any port 8080 proto tcp   # Traefik dashboard
sudo ufw allow from 192.168.1.0/24 to any port 9443 proto tcp   # Portainer UI
sudo ufw --force enable
sudo ufw status verbose
```

> ⚠️ Docker by default bypasses `ufw` by writing its own `iptables` rules. If you need stricter control, set `iptables: false` in `/etc/docker/daemon.json` **and** be prepared to maintain rules yourself. For a homelab behind NAT this is usually overkill.

---

## 🪜 Step 6 — Snapshot

Back in Proxmox UI: `docker-ubuntu` → **Snapshots** → **Take Snapshot** → name `docker-installed`.

---

## 🔌 Ports opened in this guide

| Port | Proto | Where | Reachable from | Notes |
|------|-------|-------|----------------|-------|
| *(none new on the host)* | | | | The Traefik / Portainer ports will be published by their containers in later guides. |
| `172.18.0.0/16` | — | Docker | Container-to-container only | Internal bridge subnet; not routable from the LAN. |

---

## ✅ Done — what's next?

➡️ **[TraefikPortainerSetupGuide.md](TraefikPortainerSetupGuide.md)** — deploy Portainer and Traefik on top of this Docker install.

---

## 🛟 Troubleshooting

| Symptom | Fix |
|---------|-----|
| 🚫 `permission denied while trying to connect to the Docker daemon socket` | You're not in the `docker` group yet, or didn't re-login after `usermod -aG docker $USER`. |
| 🌐 Containers can resolve external DNS but not each other | They're not on the same network. Confirm both are on `proxy-network` with `docker inspect <container> | grep NetworkMode`. |
| 💽 `/var/lib/docker` fills up | `docker system prune -af` then `docker volume prune`. Consider moving `/var/lib/docker` to a larger disk if it's a recurring problem. |
| 🔁 `docker compose` not found | You installed `docker.io` from Ubuntu repos. Use the official installer (`get.docker.com`) which ships the compose plugin. |
