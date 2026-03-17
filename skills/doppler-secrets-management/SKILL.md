---
name: doppler-secrets-management
description: "Expert knowledge for managing secrets with Doppler, including Railway integration. Use when: doppler, secrets, environment variables, DATABASE_URL, service tokens, env sync, secrets management."
risk: safe
source: "ReefCore Custom Skill"
date_added: "2026-03-04"
---

# Doppler Secrets Management

You are a Doppler secrets management expert for the ReefCore platform. Doppler is the **sole source of truth** for all environment variables. No secrets live in `.env` files, Git, or Railway's Variables tab.

## When to Use This Skill

Use this skill when:

- Setting up Doppler for a new Railway project or service
- Migrating secrets from `.env` files into Doppler
- Connecting Doppler to Railway so secrets auto-sync on deploy
- Diagnosing a Railway service crash caused by missing env vars (e.g., `DATABASE_URL empty`)
- Debugging Prisma `P1000: Authentication failed` errors (Cross-project networking issues)
- Validating secrets during Docker build phases vs Production runtime
- Adding new environment variables to a project
- Rotating secrets or API keys in production

## Core Architecture Rule

> **No secrets in `.env`, Git, or Railway Variables tab. Doppler injects everything.**

```text
Doppler (source of truth)
   └──► Railway Integration (auto-sync on deploy)
             └──► Service environment at runtime
```

The Railway dashboard Variables tab should always show variables sourced from Doppler, not manually set. If you see manually-set vars in Railway, that's a red flag — migrate them to Doppler.

## MCP Server Integration

Whenever possible, prefer using Model Context Protocol (MCP) servers for automating configurations and interacting with Doppler, Railway, or Prisma instead of relying purely on CLI commands or manual edits. MCP servers provide native context and specialized tools for orchestrating these infrastructure services directly.

## ReefCore Required Variables

All of these live in Doppler and sync into Railway:

| Variable | Description |
| -------- | ----------- |
| `DATABASE_URL` | PostgreSQL connection string (Railway internal: `postgres://...@reefcore-db.railway.internal:5432/reefcore`) |
| `REDIS_URL` | Redis connection string (Railway internal: `redis://reefcore-redis.railway.internal:6379`) |
| `VAULT_ADDR` | HashiCorp Vault address (`http://vault.railway.internal:8200`) |
| `VAULT_TOKEN` | Root/service token for Vault access |
| `NODE_ENV` | `production` in prod, `development` locally |
| `PORT` | `3000` (Railway auto-provides this too, but Doppler should mirror it) |
| `NEXTAUTH_SECRET` | JWT signing secret for Documenso/auth |
| `ADMIN_SECRET_TOKEN` | Admin endpoint guard token |
| `PAYPAL_CLIENT_ID` / `PAYPAL_SECRET` / `PAYPAL_WEBHOOK_ID` | PayPal integration credentials |
| `CASHNGO_API_KEY` / `CASHNGO_WEBHOOK_SECRET` | CashNGo integration credentials |
| `CIBC_MERCHANT_ID` / `CIBC_API_SECRET` | CIBC integration credentials |
| `REEFCORE_INTERNAL_SERVICE_KEY` | Internal service-to-service auth |
| `CBOB_WEBHOOK_SECRET` | CBOB webhook validation secret |
| `CORS_ALLOWED_ORIGINS` | Comma-separated list of allowed CORS origins |
| `VERCEL_API_TOKEN` / `VERCEL_TEAM_ID` | Vercel deployment credentials |
| `SANDDOLLAR_SERVICE_URL` | URL of the SandDollar service |
| `WEBHOOK_DEFAULT_CLIENT_ID` | Default tenant ID for webhook routing |

## Setting Up Doppler ↔ Railway Integration

### Step 1: Install Doppler CLI

```bash
# macOS/Linux
curl -Ls https://cli.doppler.com/install.sh | sudo sh

# Windows (via Scoop)
scoop install doppler
```

### Step 2: Authenticate

```bash
doppler login
```

### Step 3: Create Project & Config

In the Doppler dashboard (https://dashboard.doppler.com):

1. Create a new **Project** (e.g., `reef_core`)
2. Configs are auto-created: `dev`, `stg`, `prd`
3. Add all required variables (see table above) to the appropriate config

### Step 4: Connect to Railway (Dashboard Method)

1. In Railway Dashboard → Project → **Integrations** tab
2. Click **+ Add Integration** → **Doppler**
3. Authorize and select your Doppler project and config (`prd` for production)
4. Railway will now pull all variables from Doppler on each deploy

### Step 5: Connect to Railway (CLI Method)

```bash
# Generate a Doppler service token for Railway
doppler configs tokens create railway-sync --project reefcore --config prd

# Set this token as DOPPLER_TOKEN in Railway Variables
# Railway will automatically fetch all secrets on startup
```

### Step 6: Verify Sync

```bash
# Locally verify what Doppler would inject
doppler run --project reefcore --config prd -- env | grep DATABASE_URL

# In Railway: check the Variables tab — all vars should show "From Doppler"
```

## Local Development Workflow

### Project Setup

Before running applications locally, configure the Doppler CLI for the project. Access to a project's secrets is scoped to a specific directory.

```bash
# Select project and config for the current directory
doppler setup
```

**`doppler.yaml` Pre-configuration:**
You can pre-configure the project and config by creating a `doppler.yaml` file in the repository root or app folders (ideal for monorepos):

```yaml
setup:
  - project: reefcore
    config: dev
```

With this file, running `doppler setup` automatically uses the defined project and config.

### Injecting Secrets

Use the `doppler run` command to fetch the latest versions of secrets and inject them into your application process:

```bash
doppler run -- npm run dev
```

### Automatic Restarts

To automatically restart your application when secrets change in Doppler, use the `--watch` flag:

```bash
doppler run --watch -- npm run dev
```

## The `doppler.js` Pattern (ReefCore)

In `sanddollar_infustructure`, a Zod-validated config loader wraps all env var access:

```javascript
// sanddollar_infustructure/doppler.js
import { z } from 'zod';

const schema = z.object({
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string(),
  VAULT_ADDR: z.string().url(),
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.string().default('3000'),
  // ... all other vars
});

export const config = schema.parse(process.env);
```

This pattern ensures the app **fails fast at startup** with a clear error if any required variable is missing, rather than crashing later with a cryptic database error.

## Diagnosing Missing Env Var Crashes

If Railway crashes with `Error: Connection url is empty` or similar:

1. **Check Doppler integration is active** — Railway Dashboard → Project → Integrations → confirm Doppler is listed and connected (not showing an error)
2. **Check the correct config is mapped** — is `prd` connected to the `production` Railway environment?
3. **Check the var exists in Doppler** — `doppler get DATABASE_URL --project reefcore --config prd`
4. **Trigger a fresh deploy** — Doppler vars sync on deploy, so sometimes a redeploy is needed after adding a new variable

### Prisma `generate` Crashes During Docker Build

If your Railway deployment fails during the **Build** phase at `npx prisma generate` with a missing environment variable error (like `[prisma.config.ts] DATABASE_URL is not set`), this is because **Doppler secrets are NOT injected during the Docker build process**.

- Secrets only exist at container **runtime**.
- **Fix:** Ensure your `prisma.config.ts` or initialization scripts bypass strict environmental checks during the build build phase (e.g. check if `NODE_ENV === 'production'` before throwing the error, as Docker builds usually default to `development` or missing env vars).

### `Error: P1000: Authentication failed against database server`

If the container starts but crashes with `P1000` from Prisma:

- This usually means your Doppler `DATABASE_URL` is correct, but your Railway **Database is in a different Railway Project** than your API service.
- Railway's internal networks (e.g. `*.railway.internal`) are scoped **to the isolated project level**.
- **Fix:** Re-provision a PostgreSQL database *inside the exact same Railway project* as the web service. Generate the new `DATABASE_URL` and update it in Doppler.

## Anti-Patterns

### ❌ Setting vars directly in Railway Variables tab

This creates a split-brain situation. Doppler overrides these on next deploy, but it's confusing and error-prone.

### ❌ `.env` files in the repo

Violates Phase 0 security rules. All secrets must be in Doppler.

### ❌ `process.env.DATABASE_URL` without validation

Always validate via a Zod schema (the `doppler.js` pattern) so startup failures are caught early with helpful messages.

### ❌ Using `dev` Doppler config in production

Ensure the Railway integration maps to the `prd` Doppler config, not `dev`.

## ⚠️ Sharp Edges

| Issue | Severity | Solution |
| ----- | -------- | -------- |
| Doppler integration disconnects after Railway project recreation | high | Re-add the Doppler integration in Railway Integrations tab |
| New variable added to Doppler but service still crashes | medium | Trigger a manual redeploy in Railway (vars sync on deploy) |
| `DOPPLER_TOKEN` rotated but Railway not updated | critical | Immediately update the token in Railway before next deploy |
| Doppler dashboard shows Railway integration as "Error" | high | Re-authorize the integration — usually an OAuth token expiry |
| `DATABASE_URL` fails to initialize Prisma 7 | critical | Prisma 7 requires driver adapters. Use `DATABASE_URL` to create a `pg.Pool`, instantiate `@prisma/adapter-pg`, and pass it to `{ adapter }` in `PrismaClient`. |

## Related Skills

Works well with: `railway-build-and-deploy`, `docker-expert`, `api-security-best-practices`
