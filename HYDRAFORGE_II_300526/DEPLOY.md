# Deployment Guide

This is a **Bun server** app — a single process that serves both the Hono API (`/api/*`) and the React frontend from `dist/`. It needs a platform that runs persistent processes, not serverless.

**Recommended platforms (in order):**
1. **Railway** — easiest, auto-detects Dockerfile, ~$5/mo
2. **Fly.io** — more control, generous free tier
3. **Render** — simple but slower cold starts on free tier
4. **DigitalOcean App Platform** — solid, slightly more setup

---

## Required Environment Variables

Set all of these on your chosen platform before deploying:

```
NODE_ENV=production
PORT=3000

# Turso (your existing remote DB — same values from .env)
DATABASE_URL=libsql://your-db.turso.io
DATABASE_AUTH_TOKEN=your-turso-auth-token

# Auth
BETTER_AUTH_SECRET=<generate: openssl rand -base64 32>
WEBSITE_URL=https://your-deployed-domain.com

# OpenRouter / AI Gateway
OPENROUTER_API_KEY=your-openrouter-key
# (any other AI_GATEWAY / model keys from your .env)

# Autumn (billing) — if used
AUTUMN_SECRET_KEY=your-autumn-key

# Cloudflare R2 (file uploads) — if used
S3_ENDPOINT=https://...r2.cloudflarestorage.com
S3_BUCKET=your-bucket
S3_ACCESS_KEY_ID=...
S3_SECRET_ACCESS_KEY=...
```

> **WEBSITE_URL** must match your deployed domain exactly — better-auth uses it to validate cookies and redirects. Update it after you know the URL.

---

## Option 1 — Railway (recommended)

### First deploy
```bash
# Install Railway CLI
npm install -g @railway/cli

# Login
railway login

# From the repo root
cd /path/to/ma-review
railway init          # creates a new project
railway up            # builds Dockerfile and deploys
```

### Set env vars
In Railway dashboard → your service → Variables → paste all vars from the table above.

Or via CLI:
```bash
railway variables set DATABASE_URL="libsql://..." DATABASE_AUTH_TOKEN="..." BETTER_AUTH_SECRET="..." WEBSITE_URL="https://your-app.up.railway.app"
```

### Custom domain
Railway dashboard → Settings → Networking → Add custom domain.
Then update `WEBSITE_URL` to your custom domain.

### Subsequent deploys
```bash
railway up
```
Or connect your GitHub repo in the Railway dashboard for automatic deploys on push.

---

## Option 2 — Fly.io

### First deploy
```bash
# Install flyctl
curl -L https://fly.io/install.sh | sh

# Login
fly auth login

# From the repo root — this reads fly.toml
fly launch --no-deploy        # review/confirm config
fly secrets set \
  DATABASE_URL="libsql://..." \
  DATABASE_AUTH_TOKEN="..." \
  BETTER_AUTH_SECRET="$(openssl rand -base64 32)" \
  OPENROUTER_API_KEY="..." \
  WEBSITE_URL="https://hydraforge-ma-review.fly.dev"

fly deploy
```

### Custom domain
```bash
fly certs add yourdomain.com
# Point your DNS CNAME → hydraforge-ma-review.fly.dev
```

### Scale up if needed
```bash
fly scale vm shared-cpu-1x --memory 1024   # 1GB RAM for large PDF parsing
```

---

## Option 3 — Render

1. New Web Service → connect your GitHub repo
2. **Build Command:** `cd packages/web && bunx vite build`
3. **Start Command:** `cd packages/web && bun src/server.ts`
4. **Environment:** add all vars from the table above
5. **Runtime:** Docker (select Dockerfile) or Node 20+ with Bun installed

Render auto-detects the `Dockerfile` if you leave build/start blank.

---

## Option 4 — DigitalOcean App Platform

1. Create App → GitHub source
2. Select **Dockerfile** as build method
3. Set port to `3000`
4. Add all env vars in the Environment section
5. Deploy

---

## Database migrations on deploy

Your DB is already on Turso remote — no migration needed for a fresh deploy. If you've added schema changes since last migration:

```bash
# Run from local with production DATABASE_URL pointed at Turso
cd packages/web
bun --env-file=../../.env run db:migrate
```

Or add it as a pre-deploy step in Railway/Render's build command:
```
cd packages/web && bun --env-file=../../.env run db:migrate && bunx vite build
```

---

## Health check

All platforms can ping:
```
GET /api/health → 200 { "status": "ok" }
```

This is already configured in `railway.toml` and `fly.toml`.

---

## Troubleshooting

**Auth redirects going to wrong URL:** `WEBSITE_URL` doesn't match the deployed domain. Update the env var and redeploy.

**PDF parsing OOM:** Large contracts can use 500MB+ RAM during parse. On Fly: `fly scale vm shared-cpu-1x --memory 1024`. On Railway: bump to Starter plan.

**`DATABASE_AUTH_TOKEN` missing:** Turso remote DBs require a token even for read. Generate one at turso.tech → your database → Create token.

**Cold start on Render free tier:** Free tier spins down after 15min inactivity — first request after sleep takes ~30s. Upgrade to paid or use Railway/Fly which keep the process alive.
