# Multi-Environment Deployment Guide

This document captures the configuration patterns and pitfalls discovered while deploying a Vite SPA behind multiple hosting environments. It is most relevant for **Vite (or any SPA) apps that bake asset paths at build time** — static apps served directly by nginx usually don't need this level of care.

## Architecture Overview

| Environment | URL | Hosting |
|---|---|---|
| GitHub Pages | `https://user.github.io/<repo>/` | GitHub Actions |
| Traefik Reverse Proxy | `https://domain.com/<repo>/` | Docker + Traefik with StripPrefix |
| Direct Container Access | `http://IP:PORT/<repo>/` | Docker + nginx |

The same Docker image must serve all three without modification.

## Key Concept: Base Path Matters

Vite bakes `base` into all asset URLs at **build time**. If `base` is wrong, the browser requests assets from the wrong path — they either 404 or hit the wrong backend returning `text/plain` MIME type (blank page, no visible error).

### Rule

**Set `base` to the path users see in the browser.**

If the app is served at `domain.com/myapp/`, then `base` must be `/myapp/`. This is true even when Traefik strips the prefix before forwarding to the container — the browser still needs the full path in the HTML.

## Configuration Details

### vite.config.js

```js
base: process.env.VITE_BASE_PATH || (process.env.GITHUB_ACTIONS ? '/myapp/' : '/'),
```

- `VITE_BASE_PATH` takes priority (used by Docker)
- Falls back to `/myapp/` when building in GitHub Actions
- Default `/` for local dev (Vite dev server auto-handles paths)

> **Note:** If you rename the repo or change the subfolder path, the `GITHUB_ACTIONS` fallback must be updated to match.

### Dockerfile

```dockerfile
# Build stage
ENV VITE_BASE_PATH=/myapp/
# Must match the path Traefik routes on

# Nginx stage — support both /myapp and / access
server {
    listen 80;
    root /usr/share/nginx/html;

    # Direct container access via /myapp/
    location /myapp {
        alias /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /myapp/index.html;
    }

    # Root access (SPA fallback)
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

> **`alias` vs `root`:** The `root` directive appends the URL path to the file path, so `/myapp/index.html` would look for `/usr/share/nginx/html/myapp/index.html` which doesn't exist. Use `alias` for the subpath block.

### docker-compose.yml (Traefik labels)

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.myapp.middlewares=auto-strip-prefix@file"
  - "traefik.http.services.myapp.loadbalancer.server.port=80"
```

The `auto-strip-prefix@file` middleware is an external Traefik file-provider config that strips `/myapp` before forwarding to the container.

### GitHub Pages (deploy.yml)

No `VITE_BASE_PATH` override needed. The `GITHUB_ACTIONS` env var is automatically `true`, so `vite.config.js` falls back to `/myapp/`.

### Self-Hosted Deployment (homelabdeploy.yml)

For apps deployed via the Portainer API workflow, the Docker image is built locally on the runner and the stack is registered in Portainer. No separate image push to a registry is needed. See [homelabdeploy.yml](.github/workflows/homelabdeploy.yml) and the "Adding a new app" section in the README.

## Common Pitfalls

### 1. Base Path = `/` with Traefik StripPrefix

**Symptom:** Blank page, MIME type errors (`text/plain` instead of `application/javascript`)

**Cause:** Asset URLs resolve to `/assets/...` instead of `/myapp/assets/...`. The browser bypasses Traefik's `/myapp` route and hits the default backend, which serves files as `text/plain`.

**Fix:** Set `VITE_BASE_PATH=/myapp/` so asset URLs include the prefix.

### 2. nginx Missing Alias for Subpath

**Symptom:** 404 on direct IP access via `/myapp/`

**Cause:** nginx `root` directive appends the URL path to the file path, so `/myapp/index.html` looks for `/usr/share/nginx/html/myapp/index.html` which doesn't exist.

**Fix:** Use `alias` instead of `root` for the `/myapp` location block.

### 3. SPA Routing Fails

**Symptom:** Refreshing a sub-route returns 404

**Cause:** nginx doesn't route unknown paths back to `index.html`

**Fix:** `try_files $uri $uri/ /index.html` for root, `try_files $uri $uri/ /myapp/index.html` for the subpath block.

### 4. Cloudflare Insights CORS Errors

These are unrelated to the app configuration — they come from Cloudflare's beacon script with SRI integrity mismatches. Safe to ignore or remove if not using Cloudflare Analytics.

## Checklist for New Projects

When repeating this setup for a new project:

1. **vite.config.js** — Set `base` using `VITE_BASE_PATH` env, fallback to `/<repo-name>/` in CI
2. **Dockerfile** — Set `VITE_BASE_PATH=/<repo-name>/`, configure nginx with `alias` for the subpath
3. **docker-compose.yml** — Update container name, Traefik router/service name, and label references
4. **Traefik file provider** — Add or update the StripPrefix middleware for the new path
5. **GitHub Actions deploy.yml** — Update the `GITHUB_ACTIONS` fallback basename if the repo name changed
6. **homelabdeploy.yml** — Ensure `ComposeFile` matches the actual filename (`docker-compose.yml`, not `.yaml`)
7. **Test all three URLs** before considering the setup complete:
   - GitHub Pages URL
   - Traefik domain URL
   - Direct IP:PORT URL