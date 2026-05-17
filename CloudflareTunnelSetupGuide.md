# ☁️ Cloudflare Tunnel Setup Guide

> Create a Cloudflare Zero Trust Tunnel and run `cloudflared` on the dedicated VM so that public traffic reaches Traefik **without opening any inbound ports** on your router.

After this guide, `https://yourdomain.com/<any-path>` will reach Traefik and be routed to the right container.

---

## 📋 Prerequisites

- Completed [ProxmoxVMSetupGuide.md](ProxmoxVMSetupGuide.md) and [DockerInstallSetupGuide.md](DockerInstallSetupGuide.md).
- A registered domain on **Cloudflare** (free plan is enough). The domain must already be using Cloudflare DNS (orange-cloud).
- Traefik is *not* required to be running yet — but its port (`80` on `docker-ubuntu`) is the one we'll point the tunnel at.
- SSH access to the `cloudflared` VM (192.168.1.13).

---

## 🪜 Step 1 — Create a Cloudflare account and enable Zero Trust

1. Log in at <https://dash.cloudflare.com>.
2. Pick your domain, then in the left sidebar choose **Zero Trust**.
3. The first time you enter Zero Trust, Cloudflare asks you to create a **Team name**. Pick something like `mylab`. The free plan covers up to 50 users — plenty for a homelab.

> 📸 *Screenshot placeholder: Zero Trust dashboard home, team name set.*

---

## 🪜 Step 2 — Create the tunnel

1. In Zero Trust: **Networks → Tunnels → Create a tunnel**.
2. Connector type: **Cloudflared**.
3. Name: `homelab-tunnel`.
4. Click **Save tunnel**.
5. On the next screen, Cloudflare shows install commands for many platforms. **Copy the Linux install command** — it contains the tunnel token (a long opaque string). Keep this screen open.

> 📸 *Screenshot placeholder: Tunnel install screen with the `cloudflared service install <TOKEN>` command highlighted.*

> 🪪 The token is **a credential**. Don't paste it into screenshots, screen recordings, or commits.

---

## 🪜 Step 3 — Install `cloudflared` on the dedicated VM

SSH into the `cloudflared` VM:

```bash
ssh ubuntu@192.168.1.13
```

Install the package:

```bash
# Add Cloudflare's apt repo
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \
  | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] \
https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update
sudo apt -y install cloudflared
```

Register the tunnel as a systemd service using the token from Step 2:

```bash
sudo cloudflared service install <PASTE-TOKEN-HERE>
```

Verify the daemon is up:

```bash
sudo systemctl status cloudflared --no-pager
# Expected: active (running)
```

Back in the Cloudflare dashboard, the tunnel should now show **HEALTHY** with one connector.

> 📸 *Screenshot placeholder: Tunnel detail page showing one healthy connector.*

---

## 🪜 Step 4 — Add public hostnames

Still on the tunnel page, switch to the **Public Hostname** tab and click **Add a public hostname**.

### Option A — Wildcard (recommended)

| Field | Value |
|-------|-------|
| Subdomain | `*` |
| Domain | `yourdomain.com` |
| Path | *(leave blank)* |
| Service / Type | `HTTP` |
| Service / URL | `192.168.1.12:80` *(the `docker-ubuntu` IP)* |

This catches **everything** — `yourdomain.com`, `app1.yourdomain.com`, `anything.yourdomain.com` — and forwards it to Traefik on the LAN. Traefik then decides which container handles which path.

### Option B — Root + wildcard (some setups require both)

Add two entries: one with subdomain blank (root) and one with `*`, both pointing to the same `192.168.1.12:80`.

> 📸 *Screenshot placeholder: "Add a public hostname" form filled in with wildcard subdomain.*

> 💡 **TLS:** the public-facing URL will be HTTPS (Cloudflare handles certs). The *origin* (Traefik) speaks plain HTTP — that's fine because the LAN is trusted. If you ever move the origin off-LAN, switch the service type to `HTTPS` and provide a cert.

---

## 🪜 Step 5 — Verify end-to-end

From any device outside your LAN (phone on cellular is easiest):

```bash
curl -I https://yourdomain.com
```

Expected: a response from Traefik (probably a 404 if no app is deployed yet — that's fine; it means routing works).

> 🟢 A 404 from Traefik = success. A Cloudflare error 1033 / 530 = the tunnel can't reach the origin (check `cloudflared` is running and `docker-ubuntu:80` is open).

---

## 🪜 Step 6 — (Optional) Lock down with Cloudflare Access

If you want **email-OTP / Google / GitHub login** in front of an app before requests even reach Traefik:

1. Zero Trust → **Access → Applications → Add an application → Self-hosted**.
2. Subdomain: `admin`, Domain: `yourdomain.com`, Path: `/portainer` (for example).
3. Add a policy: **Allow** for emails matching `you@yourmail.com`.
4. Save. Now anyone hitting `https://admin.yourdomain.com/portainer` must authenticate first.

This is fantastic for putting an extra gate in front of management UIs you do choose to expose.

---

## 🔌 Ports opened in this guide

| Port | Proto | Where | Reachable from | Notes |
|------|-------|-------|----------------|-------|
| `7844` (out) | UDP/TCP | `cloudflared` VM → Cloudflare edge | Outbound to internet | The tunnel control plane. **Outbound only.** |
| *(no inbound)* | | | | The whole point of this guide is that we open **zero** inbound ports. |

If your firewall is strict, ensure outbound `443/tcp` and `7844/udp` to `*.cloudflareaccess.com` and `*.argotunnel.com` are allowed.

---

## ✅ Done — what's next?

➡️ **[GitHubRunnerSetupGuide.md](GitHubRunnerSetupGuide.md)** — register the self-hosted runner so commits trigger automatic deployments.

---

## 🛟 Troubleshooting

| Symptom | Fix |
|---------|-----|
| 🌐 Cloudflare returns **Error 1033** ("Argo Tunnel error") | `cloudflared` isn't connected. `sudo systemctl status cloudflared` and `journalctl -u cloudflared -n 50`. |
| 🌐 Cloudflare returns **Error 530 / 502** | Tunnel is up but can't reach the origin. Verify `curl -I http://192.168.1.12:80` works from the `cloudflared` VM. |
| 🔒 DNS still serves the old A record | The tunnel creates/updates CNAME records. If a stale `A` record exists in Cloudflare DNS, delete it. |
| 🚫 Public hostname saved but `curl` returns Cloudflare login page | You have Access in front of it; either authenticate or remove the policy. |
| 🪪 You lost the tunnel token | In the Cloudflare dashboard, open the tunnel → "Configure" → re-copy the install command. Old tokens stop working when you do this. |
