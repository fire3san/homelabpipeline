# ⚙️ GitHub Self-Hosted Runner Setup Guide

> Deploy the `myoung34/github-runner` Docker image on `docker-ubuntu` so that pushes to this repo automatically build and redeploy app containers — without ever opening an inbound port.

After this guide, every `git push` (or merged PR) that touches an app stack will be picked up by the runner and deployed via Docker Compose / Portainer.

**Why Docker-based?** The `myoung34/github-runner` image runs the official GitHub Actions runner inside a container. This means:
- No manual download/configure cycle — just `docker compose up -d`.
- Upgrading is a single image pull.
- The runner has Docker socket access so it can build images and run `docker compose` natively.
- No systemd service to manage — Docker handles restarts.

---

## 📋 Prerequisites

- Completed [DockerInstallSetupGuide.md](DockerInstallSetupGuide.md) — the runner needs Docker.
- SSH access to `docker-ubuntu` (192.168.1.12) as `ubuntu`.
- Admin access to the GitHub repository.

---

## 🪜 Step 1 — Get a runner registration token

1. Open your repo on GitHub: `https://github.com/<you>/homelabpipeline`.
2. Go to **Settings → Actions → Runners → New self-hosted runner**.
3. Choose **Linux** / **x64**.
4. The page shows a one-time **registration token** in the `./config.sh` command. Copy just the token value (the part after `--token`).

> 🪪 The registration token is short-lived (≈ 1 hour). Re-generate it if it expires.

---

## 🪜 Step 2 — Create the `.env` file

SSH into `docker-ubuntu`:

```bash
ssh ubuntu@192.168.1.12
```

Create the stack directory and `.env` file:

```bash
sudo mkdir -p /opt/stacks/github-runner
cd /opt/stacks/github-runner
```

Create `.env` with your token and repo URL:

```bash
cat > .env << 'EOF'
RUNNER_TOKEN=<paste-your-token-here>
RUNNER_REPOSITORY_URL=https://github.com/<you>/homelabpipeline
EOF
```

> 🔒 The `.env` file contains a secret — make sure it is `.gitignore`'d if you version this directory. The token is only needed on first start; after registration the runner persists credentials in a Docker volume.

---

## 🪜 Step 3 — Deploy the runner

Copy the compose file from this repo (or clone it):

```bash
# If you already cloned homelabpipeline:
cp ~/homelabpipeline/stacks/github-runner/docker-compose.yml .

# Otherwise, create it manually — see stacks/github-runner/docker-compose.yml
```

Bring it up:

```bash
docker compose up -d
```

Check the logs:

```bash
docker compose logs -f
```

You should see the runner register with GitHub and start idling:

```
 Runner connection is good
 Runner is listening for jobs
```

Refresh the GitHub **Settings → Actions → Runners** page — the runner should show **Idle** with a green dot.

---

## 🪜 Step 4 — Verify Docker access

The runner container mounts the Docker socket so it can build images and run compose. Verify:

```bash
docker exec github-runner docker ps
```

This should list the running containers on the host. If you get a permission error, confirm the `ubuntu` user (or whichever user Docker is configured for) has socket access.

---

## 🪜 Step 5 — Test the pipeline

Push a trivial change to `main` (e.g. edit a comment in a stack file) and check the **Actions** tab on GitHub. Within seconds you should see the job picked up by your runner.

If you are using the infra deploy workflow (`deploy.yml`), the runner will detect the changed stack and run `docker compose up -d --build` for it.

If you are using the app repo template (`templates/homelabdeploy.yml`), the runner will build the Docker image locally and register/update the Portainer stack.

---

## 🪜 Step 6 — Hardening

| Risk | Mitigation |
|------|------------|
| Untrusted PRs running on your hardware | In repo Settings → Actions → "Fork pull request workflows", set to **Require approval for all outside collaborators**. **Never** run untrusted PRs on a self-hosted runner without review. |
| Runner has full Docker access (= root on host) | Keep the repo **private** if it contains anything sensitive, or run the runner in a separate VM / Docker context. |
| Runner token in `.env` file | `chmod 600 .env` and add `.env` to `.gitignore`. The token is only used once; after registration it can be removed. |
| Runner stuck on old version | `docker compose pull && docker compose up -d` — the `:latest` tag will pull the newest image. |
| Runner container crashes | `docker compose restart`. If it persists, remove the named volume with `docker compose down -v` and re-register with a fresh token. |

---

## 🔌 Ports opened in this guide

| Port | Proto | Where | Reachable from | Notes |
|------|-------|-------|----------------|-------|
| *(no inbound)* | | | | The runner polls GitHub **outbound** on `443/tcp`. No firewall changes needed. |
| `443` (out) | TCP | runner container → `github.com`, `*.actions.githubusercontent.com` | Outbound to internet | Long-poll for jobs + artifact upload. |

---

## ✅ Done — what's next?

You've completed the core pipeline. 🎉

➡️ **[PrometheusGrafanaPVEExporterSetupGuide.md](PrometheusGrafanaPVEExporterSetupGuide.md)** — set up monitoring, or go straight to the [README](README.md#-adding-a-new-app) to deploy your first app.

---

## 🛟 Troubleshooting

| Symptom | Fix |
|---------|-----|
| ❌ Runner shows **Offline** | `docker ps` to confirm the container is running. Restart with `docker compose up -d`. Check logs with `docker compose logs -f`. |
| 🔑 `_TOKEN` errors in logs | The token expired before the container started. Generate a new one from Settings → Actions → Runners and update `.env`. |
| 🐳 Workflow fails with `permission denied on /var/run/docker.sock` | The container's user can't access the host Docker socket. Confirm the socket is mounted and the host user group is correct. |
| 🔄 Workflow runs but never picks up new commits | Check the workflow's `paths:` filter — it may be excluding your changes. Also confirm the runner labels match (`self-hosted,linux,x64,docker`). |
| 🪪 Lost the registration token | Generate a fresh one from Settings → Actions → Runners. Stop the container, delete the volume (`docker compose down -v`), update `.env`, and `docker compose up -d` again. |
| 🐢 Runner is slow to pick up jobs | This is normal — the runner uses long-polling with ≈30s latency. If it's consistently > 60s, check network connectivity to `github.com`. |