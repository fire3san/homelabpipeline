# ⚙️ GitHub Self-Hosted Runner Setup Guide

> Register a GitHub Actions self-hosted runner on `docker-ubuntu` so that pushes to this repo automatically build and redeploy app containers — without ever opening an inbound port.

After this guide, every `git push` (or merged PR) that touches an app stack will be picked up by the runner and deployed via Docker Compose / Portainer.

---

## 📋 Prerequisites

- Completed [DockerInstallSetupGuide.md](DockerInstallSetupGuide.md) — the runner needs Docker.
- SSH access to `docker-ubuntu` (192.168.1.12) as `ubuntu`.
- Admin access to the GitHub repository.

---

## 🪜 Step 1 — Register the runner on GitHub

1. Open your repo on GitHub: `https://github.com/<you>/homelabpipeline`.
2. Go to **Settings → Actions → Runners → New self-hosted runner**.
3. Choose **Linux** / **x64**.
4. GitHub will show a block of shell commands containing a one-time **registration token**. Keep the page open — you'll paste these into SSH.

> 📸 *Screenshot placeholder: "Create self-hosted runner" page with the Linux/x64 commands visible.*

> 🪪 The registration token is short-lived (≈ 1 hour). Re-generate it if it expires.

---

## 🪜 Step 2 — Install the runner on `docker-ubuntu`

SSH in:

```bash
ssh ubuntu@192.168.1.12
```

Create a dedicated directory and run **exactly the commands GitHub showed you**. They look like this (do NOT copy this verbatim — use the ones from your own repo page so the token is valid):

```bash
# Create a folder
mkdir -p ~/actions-runner && cd ~/actions-runner

# Download the latest runner package (version may differ from what GitHub shows)
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.319.1/actions-runner-linux-x64-2.319.1.tar.gz

# Extract
tar xzf ./actions-runner-linux-x64.tar.gz

# Configure (use the URL + token from the GitHub page)
./config.sh --url https://github.com/<you>/homelabpipeline --token <REG-TOKEN>
```

The configure script will ask:

| Prompt | Suggested answer |
|--------|------------------|
| Runner group | press Enter (default) |
| Name of runner | `docker-ubuntu` |
| Labels | `self-hosted,linux,x64,docker` |
| Work folder | press Enter (`_work`) |

> 📸 *Screenshot placeholder: `./config.sh` finished with "Settings Saved." and the runner appears as **Idle** in GitHub's UI.*

---

## 🪜 Step 3 — Install as a systemd service

The runner shouldn't depend on your SSH session staying open:

```bash
cd ~/actions-runner
sudo ./svc.sh install ubuntu        # runs as the `ubuntu` user
sudo ./svc.sh start
sudo ./svc.sh status
```

Confirm via systemd:

```bash
systemctl status 'actions.runner.*' --no-pager
```

Refresh the GitHub Runners page — the runner should now show **Idle** with a green dot.

---

## 🪜 Step 4 — Give the runner the tools it needs

The runner runs as the `ubuntu` user. That user must be able to:

- Run `docker` and `docker compose` (already true if you did Step 2 of the Docker guide).
- Read this repo (it's cloned automatically by Actions).
- Optionally call the Portainer API.

Quick sanity check:

```bash
sudo -u ubuntu docker ps
sudo -u ubuntu docker compose version
```

Both should succeed without `sudo`.

### (Optional) Pre-install commonly used tools

```bash
sudo apt -y install jq yq curl git unzip
```

---

## 🪜 Step 5 — Add the deploy workflow

A starter workflow lives at [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml). It:

- Triggers on `push` to `main` under `stacks/**`.
- Runs on `runs-on: [self-hosted, linux, docker]`.
- Performs `docker compose up -d --build` for the affected stack.

Push a trivial change (touch a comment) and watch the **Actions** tab on GitHub — within seconds you should see the job pick up on your `docker-ubuntu` runner.

> 📸 *Screenshot placeholder: GitHub Actions run page showing job "Deploy stack" succeeded on `docker-ubuntu`.*

---

## 🪜 Step 6 — Hardening

| Risk | Mitigation |
|------|------------|
| Untrusted PRs running on your hardware | In repo Settings → Actions → "Fork pull request workflows", set to **Require approval for all outside collaborators**. **Never** run untrusted PRs on a self-hosted runner without review. |
| Runner has full Docker access (= root) | Keep the repo **private** if it contains anything sensitive, or use a separate VM per environment. |
| Runner token in shell history | `history -d` the line, or use `read -s` to enter the token. |
| Runner stuck on old version | `cd ~/actions-runner && sudo ./svc.sh stop && ./config.sh remove --token <DEL-TOKEN>` and reinstall. |

---

## 🔌 Ports opened in this guide

| Port | Proto | Where | Reachable from | Notes |
|------|-------|-------|----------------|-------|
| *(no inbound)* | | | | The runner polls GitHub **outbound** on `443/tcp`. No firewall changes needed. |
| `443` (out) | TCP | runner → `github.com`, `*.actions.githubusercontent.com` | Outbound to internet | Long-poll for jobs + artifact upload. |

---

## ✅ Done — what's next?

You've completed the full pipeline. 🎉

➡️ Back to the [README](README.md#-adding-a-new-app) to deploy your first app, or open [`stacks/example-app/docker-compose.yml`](stacks/example-app/docker-compose.yml) as a starting point.

---

## 🛟 Troubleshooting

| Symptom | Fix |
|---------|-----|
| ❌ Runner shows **Offline** | `sudo systemctl status actions.runner.*` and `journalctl -u actions.runner.* -n 100`. Restart with `sudo ./svc.sh start`. |
| 🔑 `./config.sh` says "Http response code: 'NotFound'" | URL is wrong (must be repo-level for repo runners, org-level for org runners). |
| 🐳 Workflow fails with `permission denied on /var/run/docker.sock` | The `ubuntu` user isn't in the `docker` group, or the runner is running as a different user. |
| 🔄 Workflow runs but never picks up new commits | Check the workflow's `paths:` filter — it may be excluding your changes. |
| 🪪 Lost the registration token | Generate a fresh one from the same Settings → Actions → Runners page. |
