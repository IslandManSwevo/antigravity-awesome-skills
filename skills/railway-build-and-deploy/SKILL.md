---
name: railway-build-and-deploy
description: "Expert knowledge for building and deploying applications using the Railway CLI. Use when: railway, deploy, build, hosting, production, environment variables."
risk: safe
source: "ReefCore Custom Skill"
date_added: "2026-03-04"
---

# Railway Build and Deploy

You are a Railway deployment expert. You understand the Railway CLI, platform capabilities, and best practices for managing cloud-native deployments.

## When to Use This Skill

Use this skill when:

- Deploying code to Railway using the CLI.
- Managing environment variables for Railway services.
- Configuring build settings (Railpack, Dockerfiles, Root Directories).
- Fine-tuning deployments with watch paths and custom build/install commands.
- Integrating Railway deployments into CI/CD pipelines.
- Troubleshooting deployment failures or monitoring logs.

Your core principles:

1. **`railway up`** - The primary command for deploying code from the current directory.
2. **Environment Isolation** - Keep dev, staging, and production environments distinct.
3. **CI/CD Automation** - Use `RAILWAY_TOKEN` for non-interactive deployments.
4. **Build Optimization** - Leverage Railpack (default) or optimized Dockerfiles. Nixpacks is deprecated.
5. **Service Targeting** - Use `--service` and `--environment` flags to target specific deployments.
6. **Granular Control** - Use `rootDirectory` for monorepos and `watchPaths` to prevent unnecessary builds.

## Capabilities

- railway-cli
- deployment-automation
- cloud-native-hosting
- environment-variable-management
- log-streaming

## Requirements

- Railway CLI installed.
- Valid `RAILWAY_TOKEN` or authenticated session for CLI operations.

## Build Configuration

### ⚠️ CRITICAL: Dockerfile vs Railpack Builder

**Railway defaults to Railpack (not Dockerfile), even if a Dockerfile exists in your repo.**

This is the most common source of build failures. If your service requires a specific Node.js version, language runtime, or build tool that differs from Railpack's defaults, **you must explicitly set the builder to Dockerfile**.

**Two-step enforcement required** (both are necessary — one alone is not enough):

**Step 1: `railway.json` in repo root of the service directory:**
```json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": {
    "builder": "DOCKERFILE"
  },
  "deploy": {
    "numReplicas": 1
  }
}
```

**Step 2: Railway Dashboard** → Service → Settings → Build:
- Set **Builder** to `Dockerfile`
- Set **Dockerfile Path** to the relative path (e.g., `Dockerfile` if root is already `/bahamian-glue`)

> The dashboard setting can override `railway.json`. Always confirm both match.

### Railpack (Modern Default)

Railway uses [Railpack](https://railpack.com) by default when the builder is not explicitly set.

- **Custom Build Command**: Set in service settings or via `RAILPACK_BUILD_COMMAND`.
- **Custom Install Command**: Set via `RAILPACK_INSTALL_COMMAND`.
- **Packages**: Use `RAILPACK_PACKAGES`, `RAILPACK_BUILD_APT_PACKAGES`, or `RAILPACK_DEPLOY_APT_PACKAGES`.

### Monorepo & Root Directory

- **Root Directory**: Defaults to `/`. Change this in service settings to scoped subdirectories (e.g., `/bahamian-glue`). All build/deploy commands will operate within this root.
- **Railway Config**: `railway.json` or `railway.toml` must use absolute paths (e.g., `/backend/railway.toml`) as they don't follow the Root Directory.

### Watch Paths

Use gitignore-style patterns to trigger builds only when relevant files change.
- Example: `/packages/backend` will skip builds if only frontend files change.
- Note: Patterns always operate from `/`, even if a Root Directory is set.

## Doppler Integration (ReefCore Architecture)

ReefCore uses Doppler as the **sole source of truth** for all environment variables. Railway's Variables tab should never be manually populated.

**How it works:**
1. All secrets and env vars live in Doppler (`prd` config for production)
2. The Doppler ↔ Railway integration syncs vars automatically on each deploy
3. The service reads `process.env` as normal — Doppler injects everything

**To connect Doppler to a Railway project:**
1. Railway Dashboard → Project → Integrations → `+ Add Integration` → Doppler
2. Authorize, select project `reefcore`, config `prd`
3. Trigger a deploy — all vars will now be injected from Doppler

### ⚠️ Dockerfile Builds vs Doppler Secrets
**Doppler secrets are ONLY injected into your project's `Runtime` environment.** They are not available during Docker builds (`npm run build`, `npx prisma generate`, etc.).
- If your build fails because of a missing secret (like `DATABASE_URL` in `prisma.config.ts`), you must rewrite your config or build script to bypass strict validation until the app is actually starting up (e.g. checking `process.env.NODE_ENV === 'production'`).

### ⚠️ Cross-Project Database Networking limitations (P1000 errors)
Railway's internal networks (e.g., `*.railway.internal`) are strictly restricted horizontally to the current **Project**. 
- If you have an API in `Project A` and a Postgres DB in `Project B`, putting `Project B`'s Postgres URL into Doppler will result in a **Prisma P1000 Authentication Failed** crash at runtime because the TCP traffic is blocked.
- Databases **must** be deployed into the exact same Railway project folder as the services utilizing them for internal networking to apply.

**If a service crashes with `Error: Connection url is empty` or `DATABASE_URL is empty`:**
→ The Doppler ↔ Railway integration is not connected or the variable is missing from Doppler.
→ See the `doppler-secrets-management` skill for full diagnosis steps.

## Patterns

### Standard Code Deployment
Use `railway up` to scan, build, and deploy the current directory.
```bash
railway up --service my-service --environment production
```

### Detached Deployment (CI/CD)
Use the `--detach` flag for background deployments in automation scripts.
```bash
railway up --detach --service my-service
```

### Monorepo Partial Build

Deploy a specific service by targeting its watch path or root directory.

```bash
# Handled via Railway dashboard/config, but CLI supports context
railway up --service api-service
```

## Anti-Patterns

### ❌ Using `railway deploy` for Code
`railway deploy` is for templates/databases. Use `railway up` for custom application code.

### ❌ Hardcoding Secrets
Avoid committing secrets; use Doppler (which syncs to Railway) instead.

### ❌ Setting env vars directly in Railway Variables tab
Use Doppler exclusively. Manual Railway vars create split-brain and will be overridden.

### ❌ Deploying Untested Code to Production
Always use preview environments or staging first.

### ❌ Assuming Dockerfile is used by default
Railway uses Railpack by default. Always explicitly set `"builder": "DOCKERFILE"` in both `railway.json` AND the Railway dashboard.

## ⚠️ Sharp Edges

| Issue | Severity | Solution |
|-------|----------|----------|
| Builder set to Railpack despite having a Dockerfile | critical | Set `builder: DOCKERFILE` in `railway.json` AND confirm in Railway dashboard Settings → Build. Both must match. |
| Node.js version wrong despite `engines` field in `package.json` | high | `engines` is only respected when Railpack is the builder. If using Dockerfile, ensure `FROM node:22-alpine` (or correct version). |
| Prisma `EBADENGINE` error on `npm install` | high | Prisma 7.x requires Node ≥20.19/22.12/24.0. Switch builder to Dockerfile using `node:22-alpine`. |
| `railway.json` builder ignored | medium | Dashboard setting overrides the file. Always confirm the dashboard also shows `Dockerfile`. |
| Build fails due to missing dependencies | high | Ensure all dependencies are in `package.json` or `requirements.txt`. |
| Deployment stuck in "Building" | medium | Check `railway logs` for real-time build status. |
| Port mismatch | high | Ensure your app listens on the port provided by the `PORT` env var. |
| `RAILWAY_TOKEN` exposure | critical | Never log or commit the project token. |
| Doppler integration disconnects after project recreation | high | Re-add the Doppler integration in Railway Integrations tab. |

## Related Skills

Works well with: `docker-expert`, `doppler-secrets-management`, `vercel-deployment`, `api-security-best-practices`
