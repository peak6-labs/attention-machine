# Paperclip Cloud Run Migration Plan

> **Goal**: Move Paperclip from GCE VM (`34.71.43.100`) to Cloud Run to get an accessible `*.run.app` domain that passes Netskope AUP categorization.
>
> **Project**: `p6s-paperclip-prod` (GCP, `us-central1`)
>
> **Current**: `paperclip.paperbox.xyz` — blocked by Netskope (`[Web] Block AUP Categories` rule, `.xyz` TLD is uncategorized)

---

## Architecture: Before vs After

```
BEFORE (GCE VM)                          AFTER (Cloud Run)
─────────────────────                    ─────────────────────
Internet                                 Internet
  │                                        │
  ▼                                        ▼
Caddy (TLS, port 443)                   Cloud Run Ingress (TLS, *.run.app)
  │                                        │
  ▼                                        ▼
Paperclip Server (port 3100)             Paperclip Server (port 3100)
  │                                        │
  ▼                                        ▼
Postgres (docker-compose)                Cloud SQL (managed Postgres)
```

**What goes away**: Caddy, the GCE VM, docker-compose, `paperbox.xyz` domain, manual SSH deploys
**What you get**: `https://paperclip-<hash>-uc.a.run.app`, auto-TLS, auto-scaling, no VM to maintain

---

## Prerequisites

Before starting, verify you have:

- [ ] `gcloud` CLI installed and authenticated to `p6s-paperclip-prod`
- [ ] Docker installed locally (for building the image)
- [ ] Access to the Paperclip server Docker image (either from Artifact Registry in `p6s-paperclip-prod` or the ability to build from source)
- [ ] The current `.env` values from the VM (`/opt/paperclip/.env`)

### Check existing resources

```bash
# Verify gcloud project
gcloud config set project p6s-paperclip-prod

# Check if Artifact Registry already has Paperclip images
gcloud artifacts repositories list --location=us-central1
gcloud artifacts docker images list us-central1-docker.pkg.dev/p6s-paperclip-prod/paperclip 2>/dev/null

# Check existing Cloud SQL instances (there may already be one)
gcloud sql instances list

# Check what's running on the VM
gcloud compute ssh paperclip-prod --zone=us-central1-a \
  --command="cat /opt/paperclip/docker-compose.prod.yml"
```

---

## Step 1: Inspect Current VM State

SSH to the VM and capture everything needed for migration.

```bash
gcloud compute ssh paperclip-prod --zone=us-central1-a
```

Once on the VM:

```bash
# 1a. Capture the docker-compose config
cat /opt/paperclip/docker-compose.prod.yml

# 1b. Capture the server image name and version
sudo docker ps --format '{{.Image}} {{.Names}}'
# Expected: something like paperclipai/paperclip:latest or us-central1-docker.pkg.dev/...

# 1c. Capture environment variables (redact secrets before sharing)
cat /opt/paperclip/.env

# 1d. Capture Postgres connection details
sudo docker exec paperclip-server-1 env | grep -i postgres
# Or check docker-compose for the DB service name and credentials

# 1e. Check plugin installation path
sudo docker exec paperclip-server-1 ls -la /paperclip/x-intel-v6/

# 1f. Dump the Postgres database for migration
sudo docker exec paperclip-db-1 pg_dump -U paperclip paperclip > /tmp/paperclip-dump.sql
# (adjust user/db name based on what you find in docker-compose)
```

**Save these outputs** — you'll need them for the remaining steps.

---

## Step 2: Create Cloud SQL Instance

```bash
# 2a. Create the Postgres instance
gcloud sql instances create paperclip-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=us-central1 \
  --storage-size=10GB \
  --storage-auto-increase

# 2b. Set the postgres user password
gcloud sql users set-password postgres \
  --instance=paperclip-db \
  --password="$(openssl rand -base64 24)"
# SAVE THIS PASSWORD — you'll need it for Cloud Run env vars

# 2c. Create the database
gcloud sql databases create paperclip --instance=paperclip-db

# 2d. Get the connection name (needed for Cloud Run)
gcloud sql instances describe paperclip-db --format='value(connectionName)'
# Output: p6s-paperclip-prod:us-central1:paperclip-db
```

**Cost**: `db-f1-micro` is ~$10/month. Upgrade to `db-custom-1-3840` (~$35/month) if you need more headroom.

---

## Step 3: Migrate Data

```bash
# 3a. Copy the dump from the VM to your local machine (or Cloud Shell)
gcloud compute scp paperclip-prod:/tmp/paperclip-dump.sql /tmp/paperclip-dump.sql \
  --zone=us-central1-a

# 3b. Import into Cloud SQL
# Option A: Via gcloud (requires the dump in a GCS bucket)
gsutil cp /tmp/paperclip-dump.sql gs://p6s-paperclip-prod-backups/paperclip-dump.sql
gcloud sql import sql paperclip-db \
  gs://p6s-paperclip-prod-backups/paperclip-dump.sql \
  --database=paperclip

# Option B: Via direct psql connection (Cloud SQL Auth Proxy)
# Install: https://cloud.google.com/sql/docs/postgres/connect-auth-proxy
cloud-sql-proxy p6s-paperclip-prod:us-central1:paperclip-db &
psql -h 127.0.0.1 -U postgres -d paperclip < /tmp/paperclip-dump.sql
```

### Verify migration

```bash
psql -h 127.0.0.1 -U postgres -d paperclip -c "
  SELECT count(*) as secrets FROM secrets;
  SELECT count(*) as plugins FROM plugins;
  SELECT count(*) as plugin_entities FROM plugin_entities;
  SELECT count(*) as agents FROM agents;
"
```

---

## Step 4: Prepare the Docker Image

You need the same Docker image the VM is running. Check what image it uses (from Step 1b).

### If the image is already in Artifact Registry:

```bash
# Just note the full image URI, e.g.:
# us-central1-docker.pkg.dev/p6s-paperclip-prod/paperclip/server:latest
```

### If the image is from Docker Hub or needs rebuilding:

```bash
# Clone the Paperclip source (upstream or fork)
git clone https://github.com/peak6-labs/paperclip.git /tmp/paperclip-build
cd /tmp/paperclip-build

# Create Artifact Registry repo if needed
gcloud artifacts repositories create paperclip \
  --repository-format=docker \
  --location=us-central1

# Build and push
docker build -t us-central1-docker.pkg.dev/p6s-paperclip-prod/paperclip/server:latest .
docker push us-central1-docker.pkg.dev/p6s-paperclip-prod/paperclip/server:latest
```

---

## Step 5: Store Secrets in Secret Manager

```bash
# 5a. Secrets master key (from /opt/paperclip/.env on VM)
echo -n "YOUR_64_CHAR_HEX_KEY" | gcloud secrets create paperclip-secrets-master-key \
  --data-file=- --replication-policy=automatic

# 5b. Agent JWT secret (if used)
echo -n "YOUR_JWT_SECRET" | gcloud secrets create paperclip-agent-jwt-secret \
  --data-file=- --replication-policy=automatic

# 5c. Grant Cloud Run service account access
PROJECT_NUMBER=$(gcloud projects describe p6s-paperclip-prod --format='value(projectNumber)')
gcloud secrets add-iam-policy-binding paperclip-secrets-master-key \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
gcloud secrets add-iam-policy-binding paperclip-agent-jwt-secret \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

---

## Step 6: Deploy to Cloud Run

```bash
# Get the Cloud SQL connection name from Step 2d
CONNECTION_NAME="p6s-paperclip-prod:us-central1:paperclip-db"

# Deploy — adjust the image URI based on what you found in Step 4
gcloud run deploy paperclip \
  --image=us-central1-docker.pkg.dev/p6s-paperclip-prod/paperclip/server:latest \
  --region=us-central1 \
  --port=3100 \
  --memory=1Gi \
  --cpu=1 \
  --min-instances=1 \
  --max-instances=1 \
  --allow-unauthenticated \
  --add-cloudsql-instances="${CONNECTION_NAME}" \
  --set-env-vars="NODE_ENV=production" \
  --set-env-vars="DATABASE_URL=postgresql://postgres:YOUR_PASSWORD@localhost:5432/paperclip?host=/cloudsql/${CONNECTION_NAME}" \
  --set-secrets="PAPERCLIP_SECRETS_MASTER_KEY=paperclip-secrets-master-key:latest" \
  --set-secrets="PAPERCLIP_AGENT_JWT_SECRET=paperclip-agent-jwt-secret:latest"
```

### Key flags explained:

| Flag | Why |
|------|-----|
| `--port=3100` | Paperclip server listens on 3100 |
| `--min-instances=1` | Keep always-on (cron jobs, agent heartbeats need it) |
| `--max-instances=1` | Single instance — Paperclip isn't designed for horizontal scaling |
| `--allow-unauthenticated` | Public web UI + API (Paperclip has its own auth) |
| `--add-cloudsql-instances` | Enables Cloud SQL Auth Proxy sidecar |
| `--set-secrets` | Mounts secrets from Secret Manager as env vars |

### Note on `--min-instances=1`

Cloud Run bills per vCPU-second when instances are active. With `min-instances=1` on 1 CPU:
- ~$50-65/month (always-on)
- This is comparable to the `e2-small` VM cost

If cost is a concern, you can set `--min-instances=0` but then:
- Cold starts will delay the first request (~5-10s)
- Scheduled plugin jobs might miss their window if the container is cold
- Agent heartbeats will have latency spikes

---

## Step 7: Verify Deployment

```bash
# 7a. Get the service URL
SERVICE_URL=$(gcloud run services describe paperclip --region=us-central1 --format='value(status.url)')
echo "Paperclip URL: ${SERVICE_URL}"
# Expected: https://paperclip-<hash>-uc.a.run.app

# 7b. Health check
curl -s "${SERVICE_URL}/api/health" | jq .

# 7c. Sign in
curl -s -X POST "${SERVICE_URL}/api/auth/sign-in/email" \
  -H "Content-Type: application/json" -c /tmp/c.txt \
  -d '{"email":"zshin@peak6.com","password":"YOUR_PASSWORD"}'

# 7d. Check company exists (data migrated correctly)
curl -s -b /tmp/c.txt "${SERVICE_URL}/api/companies/7f3e0bfa-b53d-4ac2-928e-8fed4b5f310b" | jq .name

# 7e. Check agents are present
curl -s -b /tmp/c.txt "${SERVICE_URL}/api/companies/7f3e0bfa-b53d-4ac2-928e-8fed4b5f310b/agents" | jq '.[].name'
```

---

## Step 8: Reinstall the x-intelligence Plugin

Cloud Run containers are ephemeral — the plugin installed at `/paperclip/x-intel-v6` on the VM won't exist. Reinstall via the API.

```bash
# 8a. Build the plugin tarball
cd /Users/zshin/Projects/paperclip/x-intelligence-plugin
pnpm build && pnpm pack
# Produces: peak6-labs-x-intelligence-plugin-0.1.0.tgz

# 8b. Install on Cloud Run instance
# Option A: If the install API accepts a URL, host the tarball somewhere accessible
# Option B: If the install API accepts a file upload, use curl
curl -s -b /tmp/c.txt -X POST "${SERVICE_URL}/api/plugins/install" \
  -F "package=@peak6-labs-x-intelligence-plugin-0.1.0.tgz"

# 8c. Verify plugin is installed
curl -s -b /tmp/c.txt "${SERVICE_URL}/api/plugins" | jq '.[] | {id, name, status}'

# 8d. Update plugin config with company_id and secret refs
curl -s -b /tmp/c.txt -X PUT \
  "${SERVICE_URL}/api/plugins/PLUGIN_ID/config" \
  -H "Content-Type: application/json" \
  -d '{
    "company_id": "7f3e0bfa-b53d-4ac2-928e-8fed4b5f310b",
    "xai_api_key_ref": "0404f374-920f-41dd-a850-37cdb261584d",
    "x_api_bearer_ref": "afc7c87f-22a4-49c8-add2-e67e7634318f",
    "lookback_days": 3,
    "max_corpus_size": 500
  }'
```

### Important: Plugin persistence across deploys

Every time Cloud Run deploys a new revision (image update, config change), the container restarts fresh. The plugin must be reinstalled.

**Solutions (pick one):**
1. **Bake into the image** — Add the plugin tarball to the Dockerfile and run the install at build time. Best for stability.
2. **Startup script** — Add a Cloud Run startup probe that calls the install API. Works but adds cold start time.
3. **GitHub Actions post-deploy step** — After `gcloud run deploy`, run `curl` to install the plugin. Simplest for now.

---

## Step 9: Update References

After the Cloud Run deployment is verified:

### Update plugin config endpoints
All API calls in scripts and docs reference `paperclip.paperbox.xyz`. Update to the Cloud Run URL.

### Update agent API key config
OpenClaw agents have `paperclip-claimed-api-key.json` files pointing to the old URL. If the URL is embedded in the agent config, update it.

### Update the x-intelligence plugin .env (local dev)
```bash
# /Users/zshin/Projects/paperclip/x-intelligence-plugin/.env
PAPERCLIP_URL=https://paperclip-<hash>-uc.a.run.app
```

---

## Step 10: Decommission the VM (after verification)

Only after everything works on Cloud Run for at least a few days:

```bash
# Stop the VM (keeps disk, can restart if needed)
gcloud compute instances stop paperclip-prod --zone=us-central1-a

# After a week with no issues, delete it
gcloud compute instances delete paperclip-prod --zone=us-central1-a

# Delete the static IP if no longer needed
gcloud compute addresses delete paperclip-prod-ip --region=us-central1
```

---

## Rollback Plan

If Cloud Run doesn't work out:
1. The VM is still running (don't decommission until verified)
2. `paperclip.paperbox.xyz` still points to `34.71.43.100`
3. All data is in both places (VM Postgres + Cloud SQL)

To roll back: just use the VM again. Nothing is destroyed.

---

## Open Questions

These may affect the plan — investigate during execution:

1. **What Docker image is the VM running?** Need to check `docker ps` on the VM. If it's a custom build, you need the Dockerfile or source repo.
2. **Does the Paperclip server expect specific env vars beyond what's listed?** Check the full `.env` file on the VM.
3. **How does the server connect to Postgres?** The `DATABASE_URL` format in Step 6 assumes standard Postgres connection string. The actual format may differ.
4. **Does the CEO agent (`claude_local`) need to run on the same host?** If yes, it can't run on Cloud Run (no persistent shell). The CEO agent may need to stay on a VM or run elsewhere.
5. **Can the plugin install API accept tarballs via HTTP upload?** The install mechanism may require filesystem access, not HTTP upload. Check the API docs or test.
6. **What about the OpenClaw agents?** They connect to the Paperclip API URL. If that URL changes, their config needs updating too.

---

## Cost Comparison

| Component | VM (current) | Cloud Run |
|-----------|-------------|-----------|
| Compute | e2-small ~$15/mo | 1 vCPU always-on ~$55/mo |
| Database | In docker-compose (free) | Cloud SQL db-f1-micro ~$10/mo |
| TLS | Let's Encrypt (free) | Included |
| Static IP | ~$3/mo | N/A |
| Domain | paperbox.xyz ~$10/yr | Included (*.run.app) |
| **Total** | **~$19/mo** | **~$65/mo** |

Cloud Run is ~3x the cost, but eliminates: VM maintenance, SSH deploys, Caddy config, domain registration, and the Netskope block.

**Cost optimization**: If cold starts are acceptable, `--min-instances=0` drops compute to near-zero when idle (pay only for active request time). But this may break scheduled plugin jobs.

---

## Post-Migration Status (2026-03-29)

**Migration is complete.** Cloud Run is live and serving traffic.

| Component | Status |
|-----------|--------|
| Cloud Run service | Live at `https://paperclip-798386002458.us-central1.run.app` |
| Cloud SQL | Postgres 17 `paperclip-db`, data imported |
| Secret Manager | 4 secrets: master-key, better-auth-secret, anthropic-api-key, db-password |
| WIF + GitHub Actions | Configured for `peak6-labs/paperclip-deploy` |
| Plugin (x-intelligence) | v0.1.0, worker running, 5 tools + 4 jobs |
| GCE VM | Still running as rollback |

### What changed from the plan

- **Postgres 17** (not 15) — matches the VM's Postgres 17 container, avoids dump compatibility issues
- **Direct TCP to Cloud SQL** (not Unix socket) — postgres.js `new URL()` parser can't handle the `/cloudsql/` socket path format
- **2 vCPU / 2Gi** (not 1 vCPU / 1Gi) — plugin worker init needs more resources to complete within the 15s RPC timeout
- **`BETTER_AUTH_SECRET`** (not `PAPERCLIP_AGENT_JWT_SECRET`) — Paperclip uses Better Auth, not raw JWT
- **Two-repo architecture** — `paperclip-deploy` owns deployment, `x-intelligence-plugin` publishes to GitHub Packages

---

## Cleanup Checklist

### Done (2026-03-29)

- [x] Local temp files deleted (`/tmp/paperclip-db-password.txt`, `/tmp/paperclip-dump*.sql`)
- [x] Local build artifacts deleted (`.tgz` tarballs, `plugins/` dir)
- [x] Local Docker images removed
- [x] GitHub secrets set (`GCP_PROJECT_NUMBER`, `DEPLOY_TRIGGER_TOKEN`)

### Pending: Delete GitHub repos (need `delete_repo` scope)

```bash
gh auth refresh -h github.com -s delete_repo
gh repo delete peak6-labs/paperclip-old --yes
gh repo delete peak6-labs/paperclip --yes
```

### After 3 days stable (2026-04-02): Stop GCE VM

Verify Cloud Run is healthy first:
```bash
curl -sf https://paperclip-798386002458.us-central1.run.app/api/health | python3 -m json.tool
```

Then stop (not delete) the VM:
```bash
gcloud compute instances stop paperclip-prod --zone=us-central1-a --project=p6s-paperclip-prod
```

### After 7 days stable (2026-04-06): Delete GCE VM and clean GCS

```bash
# Delete the VM
gcloud compute instances delete paperclip-prod --zone=us-central1-a --project=p6s-paperclip-prod

# Delete migration dumps from GCS (data is in Cloud SQL)
gcloud storage rm gs://p6s-paperclip-prod-backups/paperclip-dump.sql
gcloud storage rm gs://p6s-paperclip-prod-backups/paperclip-dump-fixed.sql

# Optional: delete the static IP if no longer needed
gcloud compute addresses list --project=p6s-paperclip-prod
gcloud compute addresses delete <IP_NAME> --region=us-central1 --project=p6s-paperclip-prod

# Optional: delete the paperbox.xyz domain (Caddy TLS was the only use)
```
