# Postiz Production Deployment Guide (Dockploy + Neon + Redis Cloud + Cloudflare R2 + Temporal)

This guide documents the architecture, automation, and operational procedures for self-hosting Postiz in production.

---

## 1. Production Architecture Overview

The production architecture adheres strictly to YAGNI principles: external managed services are leveraged for databases and media storage to keep the VPS stateless, while Temporal and its isolated database run directly on the VPS within a private Docker bridge network without exposing unnecessary ports.

```text
                               GitHub Fork (Adi9953/postiz-app)
                                              |
                                              | git push tag vX.Y.Z
                                              v
                                  GitHub Actions (Buildx)
                                              |
                                              | docker push
                                              v
                                  GitHub Container Registry (GHCR)
                                 ghcr.io/adi9953/postiz-app:vX.Y.Z
                                              |
                                              v
                                   Dockploy on VPS Host
                                              |
                                              v
                   +------------------------------------------------------+
                   | VPS (Docker Network: postiz-internal)                |
                   |                                                      |
                   |  [Traefik Reverse Proxy (Dockploy)] :80 / :443 SSL   |
                   |                        |                             |
                   |                        v :5000                       |
                   |  [Postiz Container]                                  |
                   |    ├── Nginx (:5000 ingress)                         |
                   |    ├── Next.js Frontend (:4200)                      |
                   |    ├── NestJS Backend (:3000)                        |
                   |    └── Temporal Orchestrator Worker (:3002)          |
                   |                        |                             |
                   |                        v gRPC :7233                  |
                   |  [Temporal Server] (temporalio/auto-setup:1.28.1)    |
                   |    └── ENABLE_ES=false (SQL Advanced Visibility)     |
                   |                        |                             |
                   |                        v :5432                       |
                   |  [Temporal PostgreSQL] (postgres:16-alpine)          |
                   |    └── Volume: temporal-postgres-data                |
                   +------------------------+-----------------------------+
                                            |
                         +------------------+------------------+
                         |                  |                  |
                         v                  v                  v
                 [Neon PostgreSQL]  [Redis Cloud Free]  [Cloudflare R2]
                   Application DB       30 MB Cache       Object Storage
```

---

## 2. Temporal Investigation & Elimination of Elasticsearch

### What Services Are Required?

| Service | Status | Justification |
|---|---|---|
| **Temporal Server** | **REQUIRED** | Postiz relies on Temporal workflows for scheduling, executing, and retrying social media posts and platform automations. |
| **Temporal PostgreSQL** | **REQUIRED (on VPS)** | Dedicated persistence store for Temporal cluster execution state, task queues, and visibility schemas. Isolated to the private internal Docker network. |
| **Temporal Elasticsearch** | **REMOVED** | **Not required.** Temporal Server v1.20+ supports SQL Advanced Visibility natively on PostgreSQL 12+. `temporalio/auto-setup:1.28.1` automatically creates and migrates `temporal` and `temporal_visibility` databases when `ENABLE_ES=false`. Postiz only executes equality visibility queries (`postId="${postId}" AND ExecutionStatus="Running"`). Removing Elasticsearch saves 512 MB – 1 GB of VPS RAM and prevents OOM crashes. |
| **Temporal UI** | **REMOVED** | Developer/Admin dashboard. Postiz backend processes communicate directly with Temporal via gRPC; the web UI is not required for production operation. |
| **Temporal Admin Tools** | **REMOVED** | CLI administration container (`tctl`/`temporal`). Not required for runtime operation. |

### Port Privacy
* `temporal:7233` and `temporal-postgresql:5432` have **no published host ports**.
* Only the `postiz` container communicates with `temporal:7233` across the internal Docker bridge network (`postiz-internal`).

---

## 3. GitHub Actions & GHCR Publishing Pipeline

The workflow [`.github/workflows/build-containers.yml`](file:///Users/adi/My%20Projects/postiz/postiz-app/.github/workflows/build-containers.yml) builds and pushes production images to GitHub Container Registry (GHCR).

### When Is an Image Built?
1. **On Release Tag**: Whenever a semantic tag matching `v*.*.*` (e.g. `v1.47.0`) is pushed to GitHub:
   ```bash
   git tag v1.47.0
   git push origin v1.47.0
   ```
2. **On Manual Dispatch**: In the GitHub Actions tab, select "Build and Publish Container" and enter an optional version tag.

### GHCR Image Format
* Image format: `ghcr.io/<github-owner>/postiz-app:<version>`
* Example: `ghcr.io/adi9953/postiz-app:v1.47.0`
* A `latest` tag is also published alongside the immutable version tag for convenience.
* **Production rule**: Always deploy the immutable version tag (e.g. `v1.47.0`) in Dockploy so rollbacks are predictable and atomic.

### GHCR Permissions Setup
In your GitHub fork repository settings:
1. Go to **Settings** > **Actions** > **General** > **Workflow permissions**.
2. Select **Read and write permissions**.
3. Under your GitHub profile/organization **Packages** settings, ensure your package `postiz-app` is set to **Public** (or configure a Docker Registry credential with a GitHub Personal Access Token in Dockploy if private).

---

## 4. Dockploy Deployment Guide

### Step 1: Create a New Compose Stack in Dockploy
1. Log in to your Dockploy dashboard on your VPS.
2. Create a new **Project** (e.g. `Postiz Production`) or select an existing one.
3. Click **Add Service** and select **Docker Compose**.
4. Set the Compose source to:
   * **Repository**: Connect your GitHub fork (`Adi9953/postiz-app`).
   * **Branch**: `main`
   * **Compose Path**: `docker-compose.prod.yaml`

### Step 2: Configure Environment Variables
In the **Environment** tab of your Dockploy stack, copy the contents from [`.env.production.example`](file:///Users/adi/My%20Projects/postiz/postiz-app/.env.production.example) and replace with your real secrets:

```env
GHCR_REPO=adi9953/postiz-app
POSTIZ_VERSION=v1.47.0
POSTIZ_PORT=5000

MAIN_URL=https://postiz.yourdomain.com
FRONTEND_URL=https://postiz.yourdomain.com
NEXT_PUBLIC_BACKEND_URL=https://postiz.yourdomain.com/api
BACKEND_INTERNAL_URL=http://localhost:3000

JWT_SECRET=<generate_secure_random_64_char_string>
IS_GENERAL=true
DISABLE_REGISTRATION=false

DATABASE_URL=postgresql://[user]:[password]@[neon-host]/[dbname]?sslmode=require
REDIS_URL=redis://default:[password]@[redis-cloud-host]:[port]

STORAGE_PROVIDER=cloudflare
CLOUDFLARE_ACCOUNT_ID=[your-account-id]
CLOUDFLARE_ACCESS_KEY=[your-r2-access-key-id]
CLOUDFLARE_SECRET_ACCESS_KEY=[your-r2-secret-access-key]
CLOUDFLARE_BUCKETNAME=[your-r2-bucket-name]
CLOUDFLARE_BUCKET_URL=https://[your-r2-public-url-or-domain]
CLOUDFLARE_REGION=auto

TEMPORAL_ADDRESS=temporal:7233
TEMPORAL_POSTGRES_USER=temporal
TEMPORAL_POSTGRES_PASSWORD=<generate_secure_password>
```

> [!IMPORTANT]
> Ensure none of `MAIN_URL`, `FRONTEND_URL`, `NEXT_PUBLIC_BACKEND_URL`, or `CLOUDFLARE_BUCKET_URL` end with a trailing slash (`/`). Trailing slashes will fail Postiz's startup configuration check.

> [!IMPORTANT]
> `REDIS_URL` must start with `redis://`.

### Step 3: Configure Domain and SSL in Dockploy
1. Under **Domains** in Dockploy for the `postiz` service:
   * **Domain**: `postiz.yourdomain.com`
   * **Target Port**: `5000`
   * **HTTPS**: Enabled (handled by Dockploy's Traefik reverse proxy via Let's Encrypt).
2. Save and click **Deploy**.

---

## 5. External Services Configuration

### Neon PostgreSQL
1. Create a project on [Neon](https://neon.tech).
2. Copy the connection string.
3. Ensure the connection string includes `?sslmode=require`.
4. Use either the direct connection string or the pooled connection string (`-pooler`).
5. **Backup tip**: Use Neon's instant branching to create a snapshot branch (e.g. `backup-before-v1.48.0`) before running version upgrades.

### Redis Cloud (Free 30 MB)
1. Create a free database on [Redis Cloud](https://redis.io/cloud/).
2. Copy the public endpoint host, port, and default user password.
3. Format the connection string as:
   `redis://default:<password>@<endpoint-host>:<port>`
4. Postiz only uses Redis for session throttling, rate limits, and concurrency locks. 30 MB is well within requirements.

### Cloudflare R2
1. In Cloudflare dashboard, navigate to **R2**.
2. Create a bucket (e.g. `postiz-media`).
3. Under bucket **Settings** > **Public Access**:
   * Enable **R2.dev subdomain** OR connect a **Custom Domain** (e.g. `media.yourdomain.com`).
   * Copy the public URL (e.g. `https://pub-xxxx.r2.dev` or `https://media.yourdomain.com`). This becomes `CLOUDFLARE_BUCKET_URL`.
4. Go to **R2** > **Manage R2 API Tokens** and create an API token with **Object Read & Write** permissions for your bucket.
5. Record:
   * `Account ID` -> `CLOUDFLARE_ACCOUNT_ID`
   * `Access Key ID` -> `CLOUDFLARE_ACCESS_KEY`
   * `Secret Access Key` -> `CLOUDFLARE_SECRET_ACCESS_KEY`

---

## 6. Upstream Update Strategy

Because this repository is a clean fork with zero application modifications, syncing with upstream Postiz releases is straightforward:

```text
Upstream Release (gitroomhq/postiz-app)
                  |
                  v git fetch upstream
         Local Fork Branch
                  |
                  v git merge upstream/main
         Review & Local Test
                  |
                  v git tag v1.48.0 && git push origin v1.48.0
         GitHub Actions (GHCR Build)
                  |
                  v update POSTIZ_VERSION=v1.48.0
              Dockploy
```

### Exact Update Commands:
```bash
# 1. Fetch latest changes from official Postiz upstream
git fetch upstream

# 2. Switch to main and merge upstream
git checkout main
git merge upstream/main

# 3. Push updated code to your GitHub fork
git push origin main

# 4. Create an immutable release tag
git tag v1.48.0
git push origin v1.48.0
```

Once pushed, GitHub Actions automatically builds `ghcr.io/adi9953/postiz-app:v1.48.0`. Once the GitHub Actions workflow completes, update `POSTIZ_VERSION=v1.48.0` in Dockploy and click **Deploy**.

---

## 7. Database Migration Safety & Rollback Plan

### How Postiz Handles Database Migrations
Postiz runs `prisma db push --accept-data-loss` automatically on container startup inside `pm2-run` before starting the application services. It does not use manual migration scripts (`prisma/migrations`).
* Additive changes (new tables, new nullable columns) are automatically applied safely on startup.
* Postiz does NOT execute destructive drops unless a column was explicitly removed upstream.

### Safe Upgrade Procedure:
1. Go to Neon dashboard and create a branch from `main` (e.g. `pre-upgrade-v1.47.0`).
2. Update `POSTIZ_VERSION` to the new tag in Dockploy and redeploy.
3. Verify application health and logs.

### Rollback Procedure:
If a problem is detected after deploying a new version:
1. In Dockploy, navigate to your stack **Environment** tab.
2. Change `POSTIZ_VERSION` back to the previous version tag:
   ```env
   POSTIZ_VERSION=v1.47.0
   ```
3. Click **Deploy**. Dockploy pulls the previous image and restarts the container within seconds.
4. If database rollback is needed, switch `DATABASE_URL` to your pre-upgrade Neon branch.

---

## 8. Production Validation Checklist

Use this checklist to verify your production deployment:

```text
[ ] Docker image builds via GitHub Actions
[ ] Image pushed to GHCR under ghcr.io/<owner>/postiz-app:vX.Y.Z
[ ] Dockploy pulls image from GHCR
[ ] Postiz starts (Nginx, backend, frontend, orchestrator)
[ ] Temporal starts without Elasticsearch
[ ] Temporal PostgreSQL starts and healthcheck passes
[ ] Postiz connects to Temporal (gRPC port 7233)
[ ] Postiz connects to Neon PostgreSQL (Prisma db push completes)
[ ] Postiz connects to Redis Cloud (ioredis initializes)
[ ] R2 file uploads and avatar saves work
[ ] HTTPS works through Dockploy / Traefik
[ ] Registration / Admin account creation works
[ ] Social integration can be connected
[ ] Database migrations work automatically on startup
[ ] Container restart works cleanly
[ ] Rollback to previous version tag works
```
