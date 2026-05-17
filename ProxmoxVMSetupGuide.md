# 🖥️ Proxmox VM Setup Guide

> Provision the three Ubuntu Server VMs (`openclaw-ubuntu`, `docker-ubuntu`, `cloudflared`) that host the homelab pipeline.

This guide is the **foundation** for everything else. Plan ~2 hours from a fresh Proxmox install to three SSH-able VMs.

---

## 📋 Prerequisites

- A spare machine (mini-PC, NUC, old desktop) with:
  - ≥ 4 CPU threads, ≥ 16 GB RAM, ≥ 250 GB disk recommended.
  - x86-64 CPU with VT-x / AMD-V enabled in BIOS.
- A USB stick (≥ 4 GB) for the Proxmox installer.
- A wired Ethernet connection to your home router.
- A laptop on the same LAN for SSH access.

---

## 🪜 Step 1 — Install Proxmox VE

1. Download the latest **Proxmox VE** ISO from <https://proxmox.com/en/downloads>.
2. Flash it to USB with **Rufus** (Windows) or `dd` (Linux/macOS).
   > 📸 *Screenshot placeholder: Rufus configured with the Proxmox ISO and "DD image" mode selected.*
3. Boot the target machine from USB. Follow the installer:
   - Filesystem: `ext4` is fine for a single disk. ZFS if you have two disks for redundancy.
   - Hostname: `pve.lab` (or your preference).
   - **Static IP** on your LAN, e.g. `192.168.1.10/24`, gateway `192.168.1.1`, DNS `1.1.1.1`.
4. Reboot, remove USB, and visit `https://192.168.1.10:8006` from your laptop.
   > 📸 *Screenshot placeholder: Proxmox web UI login screen.*

> ⚠️ Proxmox uses a self-signed cert by default — browser warnings are normal. Click through.

### Post-install housekeeping

- Disable the enterprise repo (it asks for a paid subscription):
  ```bash
  # SSH in as root
  sed -i 's|^deb|# deb|' /etc/apt/sources.list.d/pve-enterprise.list
  echo 'deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription' \
    > /etc/apt/sources.list.d/pve-no-subscription.list
  apt update && apt -y dist-upgrade
  ```
- (Optional) suppress the "no valid subscription" popup — search the Proxmox forum if it annoys you.

---

## 🪜 Step 2 — Upload the Ubuntu Server ISO

1. Download **Ubuntu Server 24.04 LTS** from <https://ubuntu.com/download/server>.
2. In the Proxmox UI: `pve` → `local (pve)` → **ISO Images** → **Upload**.
   > 📸 *Screenshot placeholder: ISO upload dialog with `ubuntu-24.04-live-server-amd64.iso` selected.*

---

## 🪜 Step 3 — Create the three VMs

We'll create them one at a time. The recipe is the same for all three; only **name, IP, and resources** change.

### Common VM template

In Proxmox UI: top-right **Create VM**.

| Tab | Setting | Value |
|-----|---------|-------|
| General | Name | *(see table below)* |
| OS | ISO image | `ubuntu-24.04-live-server-amd64.iso` |
| System | BIOS | `Default (SeaBIOS)` |
| System | Machine | `q35` |
| System | Qemu Agent | ☑️ |
| Disks | Bus/Device | `VirtIO Block` |
| Disks | Size | *(see table below)* |
| Disks | Discard | ☑️ (for SSD trim) |
| CPU | Type | `host` (best performance) |
| CPU | Cores | *(see table below)* |
| Memory | Size | *(see table below)* |
| Network | Bridge | `vmbr0` |
| Network | Model | `VirtIO (paravirtualized)` |

### Per-VM sizing

| VM name | vCPU | RAM | Disk | Static LAN IP |
|---------|------|-----|------|---------------|
| `docker-ubuntu` | 4 | 8 GB | 80 GB | `192.168.1.12` |
| `openclaw-ubuntu` | 2 | 4 GB | 40 GB | `192.168.1.11` |
| `cloudflared` | 1 | 1 GB | 16 GB | `192.168.1.13` |

> 📸 *Screenshot placeholder: Create-VM wizard "Confirm" tab showing the summary for `docker-ubuntu`.*

---

## 🪜 Step 4 — Install Ubuntu Server in each VM

For **each** VM:

1. Click **Start**, then **Console**.
2. Walk through the Ubuntu Server installer:
   - Language: English.
   - Keyboard: your layout.
   - Network: choose **Edit IPv4** → **Manual**, set the static IP from the table above. Gateway `192.168.1.1`, DNS `1.1.1.1`.
   - Proxy / mirror: leave defaults.
   - Storage: **Use entire disk**.
   - Profile:
     - Server name: matches the VM name (`docker-ubuntu`, etc.).
     - Username: pick something memorable, e.g. `ubuntu` — referenced as `/home/ubuntu/...` in later guides.
     - Password: strong, store in a password manager.
   - **☑️ Install OpenSSH server.**
   - Snaps: skip.
3. Reboot when prompted.

> 📸 *Screenshot placeholder: Ubuntu installer "Profile setup" with username `ubuntu` and OpenSSH ticked.*

### Smoke test from your laptop

```bash
ssh ubuntu@192.168.1.12   # docker-ubuntu
ssh ubuntu@192.168.1.11   # openclaw-ubuntu
ssh ubuntu@192.168.1.13   # cloudflared
```

If all three respond, you're done with provisioning. 🎉

---

## 🪜 Step 5 — Baseline hardening (apply to every VM)

```bash
# Update everything
sudo apt update && sudo apt -y full-upgrade

# Install QEMU guest agent so Proxmox can shut the VM down cleanly
sudo apt -y install qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent

# Set timezone
sudo timedatectl set-timezone Asia/Hong_Kong   # adjust to taste

# Enable automatic security updates
sudo apt -y install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

### SSH hardening (recommended)

On your laptop, generate a key once if you don't have one:

```bash
ssh-keygen -t ed25519 -C "homelab"
```

Then copy it to each VM:

```bash
ssh-copy-id ubuntu@192.168.1.12
ssh-copy-id ubuntu@192.168.1.11
ssh-copy-id ubuntu@192.168.1.13
```

Now disable password auth on each VM:

```bash
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl reload ssh
```

> 🧯 **Test that key auth works** in a *second* SSH session before closing your first one. Locking yourself out is annoying.

---

## 🪜 Step 6 — Snapshot

In the Proxmox UI, select each VM → **Snapshots** → **Take Snapshot**, name it `baseline-clean`.

> 💾 You can roll back to this state in 10 seconds if anything later goes wrong.

---

## 🔌 Ports opened in this guide

| Port | Proto | Where | Reachable from | Notes |
|------|-------|-------|----------------|-------|
| `22` | TCP | All three VMs | LAN only | SSH; key-auth only after Step 5. |
| `8006` | TCP | Proxmox host | LAN only | Proxmox web UI. Never expose to internet. |

No other ports are opened by this guide. Each subsequent guide will explicitly list any new ports it adds.

---

## ✅ Done — what's next?

You now have three clean Ubuntu VMs ready for software. Continue with:

➡️ **[DockerInstallSetupGuide.md](DockerInstallSetupGuide.md)** — install Docker and create the `proxy-network` bridge.

---

## 🛟 Troubleshooting

| Symptom | Fix |
|---------|-----|
| 🌐 VM has no internet inside the installer | Check the VM's Network device is on `vmbr0`, and your router's DHCP/static IP doesn't conflict. |
| 🖥️ Console is blank / black | Click into the console window first, then press Enter. Some browsers swallow the first keystroke. |
| 🐌 Installer is very slow | Make sure CPU type is `host` and the disk bus is `VirtIO Block` (not IDE). |
| 🔑 SSH key auth fails | Re-run `ssh-copy-id`, and check `/home/ubuntu/.ssh/authorized_keys` exists with `chmod 600`. |
