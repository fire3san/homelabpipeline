# Complete Traefik & Portainer Setup Guide (Automated Subfolder Routing)

This guide details how to set up Traefik as a reverse proxy using Portainer, routing traffic through a Cloudflare tunnel automatically to Docker containers using subfolder paths (e.g., `domain.com/app`).

## Prerequisites
* Ubuntu Server with Docker installed.
* A Cloudflare Tunnel set up and pointing `*` (wildcard) or your root domain to the Ubuntu Server's IP address (Port 80).

## Step 1: Foundation & Portainer
Before deploying any stacks, you must create a global Docker network. This ensures Portainer doesn't accidentally isolate containers in stack-specific networks.

**1. Create the proxy network via SSH:**
```bash
docker network create proxy-network
```

**2. Install Portainer:**
Run this command to spin up the Portainer management interface:
```bash
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:lts
```
*Access Portainer at `https://<YOUR-SERVER-IP>:9443` and complete the initial setup.*

## Step 2: Traefik's Dynamic Configuration
Traefik needs a static file to define custom middlewares (like stripping the subfolder name before passing traffic to an app). 

**1. Create the directories via SSH:**
*(Do not use the `~` shortcut in Portainer later, use the absolute path).*
```bash
mkdir -p /home/ubuntu/traefik/config
nano /home/ubuntu/traefik/config/dynamic.yml
```

**2. Add the Regex Middleware:**
Paste the following into `dynamic.yml`. This rule slices off the first folder path (e.g., `/test-app`) so the container just receives `/`.
```yaml
http:
  middlewares:
    auto-strip-prefix:
      stripPrefixRegex:
        regex:
          - "^/[^/]+"
```

## Step 3: Deploy Traefik (The Traffic Cop)
In Portainer, go to **Stacks** -> **Add stack**. Name it `traefik`. 

> ⚠️ **CRITICAL WARNINGS:**
> * **Image Version:** Always use `traefik:latest` (or `v3.6+`). Older versions (`v3.0`) have an API mismatch with newer Docker Engines and will fail silently.
> * **Absolute Paths:** Portainer runs as root. Use `/home/ubuntu/...` for volume mounts, never `~/...`.

**Traefik `docker-compose.yml`:**
```yaml
version: '3.8'

services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: always
    command:
      - "--api.insecure=true"                           # Opens local dashboard on 8080
      - "--entrypoints.web.address=:80"                 # Cloudflare entrypoint
      - "--providers.docker=true"                       # Connect to Docker
      - "--providers.docker.exposedbydefault=false"     # Turn off auto-expose for security
      - "--providers.docker.network=proxy-network"      # Force Traefik to use our bridge
      
      # 🤖 MASTER AUTOMATION RULE
      # Automatically builds the rule: Host(domain.com) && PathPrefix(/service-name)
      # NOTE: In Portainer, you MUST define the DOMAINNAME environment variable at the bottom of the Stack page!
      - "--providers.docker.defaultRule=Host(`yourdomain.com`) && PathPrefix(`/{{ index .Labels \"com.docker.compose.service\" }}`)"      
      
      - "--providers.file.filename=/etc/traefik/config/dynamic.yml"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /home/ubuntu/traefik/config:/etc/traefik/config:ro
    networks:
      - proxy-network

networks:
  proxy-network:
    external: true
```
*Note: Ensure you add the Environment Variable `DOMAINNAME` = `yourdomain.com` in Portainer before deploying.*

## Step 4: Deploying an App
Because Traefik handles the heavy lifting, deploying an app is incredibly simple. Create a new stack for your app.

**Test App `docker-compose.yml`:**
```yaml
version: '3.8'

services:
  # 🚀 Whatever you name this service block becomes your URL path: /test-app
  test-app:
    image: nginxdemos/hello:latest
    container_name: test-container
    restart: always
    ports:
      - "8081:80" # Keeping LAN port open for local bypass/safety
    networks:
      - proxy-network
    labels:
      # 1. Turn on the porch light so Traefik sees it
      - "traefik.enable=true"
      
      # 2. Apply the dynamic.yml middleware to strip the /test-app folder prefix
      - "traefik.http.routers.test-app.middlewares=auto-strip-prefix@file"
      
      # 3. Tell Traefik which internal port the container listens on
      - "traefik.http.services.test-app.loadbalancer.server.port=80"

networks:
  proxy-network:
    external: true
```

## 🛠️ Troubleshooting Checklist
If you get a **404 Page Not Found**:
1. **Check the Traefik Dashboard:** Go to `http://<YOUR-SERVER-IP>:8080/dashboard/`. Check the **Routers** tab. If your app isn't listed, Traefik can't see it.
2. **Check the Docker Socket:** Look at Traefik's logs in Portainer. If you see `"client version is too old"`, update the Traefik image. If you see `"permission denied"`, ensure Traefik has read access to `/var/run/docker.sock`.
3. **Check Router Names:** When using automated rules, Traefik names the router after the service. Ensure your middleware label perfectly matches the service name (e.g., `traefik.http.routers.<SERVICE_NAME>.middlewares`).
4. **Check Variables:** If Portainer mangles the `${DOMAINNAME}` variable, you might need to temporarily hardcode your domain directly into the `defaultRule` inside the Traefik stack to isolate the issue.