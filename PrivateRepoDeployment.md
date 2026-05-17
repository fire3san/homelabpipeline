# Private Repository Deployment

This guide explains how to use the **Zero Touch Homelab Deployment** with **private** GitHub repositories.

## Why special handling is needed

When your app repository is **private**, Portainer cannot pull the code unless it has Git credentials.  
The public template uses `RepositoryAuthentication: false`. For private repos we enable authentication.

## 1. Add Repository Secrets (in your private app repo)

Go to your app repo → **Settings → Secrets and variables → Actions**

Add these two secrets:

| Secret Name   | Value                                      |
|---------------|--------------------------------------------|
| `GIT_USERNAME`| Your GitHub username (e.g. `fire3san`)    |
| `GIT_PAT`     | Classic Personal Access Token with **`repo`** scope |

> **Never commit these secrets!** They are only used at deployment time.

## 2. Use the Private Version in your workflow

In `.github/workflows/homelabdeploy.yml`, use the **PRIVATE REPO VERSION** section (already included and active in the latest `templates/homelabdeploy.yml`).

The template now contains **both public and private versions** side-by-side with clear comments.

## 3. First-time deployment

After pushing the workflow:
- The first run will create a new stack in Portainer using your private repo.
- All future pushes will automatically trigger a redeploy.

## Security notes

- Credentials are sent **only** to your own Portainer instance.
- The public `homelabpipeline` repo never contains any real tokens.
- Keep the full workflow with secrets **only** inside your private app repositories.

For public repositories, simply use the commented **PUBLIC REPO VERSION** instead.