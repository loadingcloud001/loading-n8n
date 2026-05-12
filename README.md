# loading-n8n

Workflow automation platform for connecting services and automating business processes at Loading Technology.

## What is n8n?

n8n is an open-source workflow automation platform that allows you to connect different services and automate business processes through a visual workflow editor. It provides:

- **Visual Workflow Editor**: Drag-and-drop interface for creating automations
- **Node Library**: 400+ pre-built integrations (GitHub, Slack, PostgreSQL, S3, etc.)
- **Webhook Handler**: Trigger workflows via HTTP requests
- **Execution Logs**: Track and debug workflow runs
- **SSO via Auth0**: Enterprise-grade single sign-on authentication

## Architecture

```
Nginx Reverse Proxy (Auth0-authenticated)
                    |
                    v
              +---------+
              |   n8n   |
              | (Docker) |
              +---------+
         +-----|-----|-----+
         v     v     v     v
    +----------+ +--------+ +---------+
    |    DO    | | Auth0  | |DO Spaces|
    |PostgreSQL| |  SSO   | |   S3    |
    +----------+ +--------+ +---------+
```

### Data Sources

| Source | Connection | Purpose |
|--------|-----------|---------|
| DO Managed PostgreSQL | PostgreSQL | Workflow and execution data |
| Auth0 | OAuth2 | Single sign-on |
| DO Spaces | S3-compatible | File storage for workflows |

### Upgrade Architecture

See [architecture.md](architecture.md) for full details. Summary:

- **Version-pinned** deployments via `N8N_VERSION` env variable
- **Automated backup** before every upgrade (pg_dump to /opt/loading-n8n/backups/)
- **Graceful shutdown** with 30s stop_grace_period for in-flight executions
- **Health check with retry** (up to 10 attempts, 6s intervals) against `/healthz`
- **Automated rollback** to previous version if health check fails

## Configuration

### Environment Variables

Copy `.env.example` to `.env` and configure:

```bash
cp .env.example .env
```

**Version:**
```bash
N8N_VERSION=latest  # Pin to specific version for production (e.g., 1.93.1)
```

**Service Settings:**
```bash
N8N_HOST=n8n.loadingtechnology.app
N8N_PORT=5678
WEBHOOK_URL=https://n8n.loadingtechnology.app
N8N_PROTOCOL=https
```

**Auth0 SSO:**
```bash
N8N_OAUTH2_CLIENT_ID=<your-client-id>
AUTH0_CLIENT_SECRET=<your-client-secret>
N8N_OAUTH2_ISSUER=https://<your-tenant>.auth0.com/
```

**Database (DO Managed PostgreSQL):**
```bash
DO_DB_HOST=private-postgres-do-user-34964671-0.i.db.ondigitalocean.com
DO_DB_PORT=25060
DO_DB_NAME=loading_n8n
DO_DB_USER=doadmin
DO_DB_PASSWORD=<your-password>
```

### SSL Certificate

The deployment uses a custom CA certificate from DigitalOcean. Ensure the certificate is mounted at:
```
/usr/local/share/ca-certificates/do-project-ca.crt -> /etc/ssl/certs/do-project-ca.crt
```

## Deployment

### Prerequisites

- Docker and Docker Compose v2 installed
- Access to `nginx-network` external network
- DO Managed PostgreSQL instance
- Auth0 application configured

### Local Development

```bash
# Start with latest version
docker compose up -d

# Start with specific version
N8N_VERSION=1.93.1 docker compose up -d
```

### GitHub Actions Deployment

Three workflows manage production deployment:

| Workflow | Trigger | Use Case |
|----------|---------|----------|
| **CI** (`ci.yml`) | Push/PR to `main` | Validates syntax, checks for leaked secrets |
| **Deploy** (`deploy.yml`) | Push to `main` or manual | Standard deployment, optional backup |
| **Upgrade** (`upgrade.yml`) | Manual dispatch | Versioned upgrade with backup + rollback |

#### Standard Deploy (push to main)

1. Push changes to `main` branch
2. `deploy.yml` runs automatically:
   - Pre-deploy DB backup
   - Graceful shutdown of current container
   - Pull + start new container
   - Health check with retry
   - Verification

#### Manual Version Upgrade

1. Go to **Actions → Upgrade n8n → Run workflow**
2. Enter target version (e.g., `1.93.1`)
3. Optionally uncheck "Skip pre-upgrade database backup"
4. Click **Run workflow**

The upgrade workflow:
1. Captures current version for rollback
2. Creates database backup (unless skipped)
3. Pulls and deploys new version
4. Runs health check with up to 10 retries
5. On failure: automatically rolls back to previous version
6. Posts summary with results

### Start the Service

```bash
docker compose up -d
```

### View Logs

```bash
docker compose logs -f n8n
```

### Restart

```bash
docker compose restart n8n
```

### Stop

```bash
docker compose down
```

## Access

- **URL**: https://n8n.loadingtechnology.app
- **Port**: 5678 (internal)

## Credentials

### Admin Account

| Field | Value |
|-------|-------|
| Email | admin@loadingtechnology.app |
| Password | AdminPassword123! |

**Note**: This account is for initial setup. SSO via Auth0 is the primary authentication method for regular use.

## Secrets

The following secrets are required for deployment:

| Secret | Description | Source |
|--------|-------------|--------|
| `DO_DB_PASSWORD` | PostgreSQL database password | DO Database Console |
| `AUTH0_CLIENT_SECRET` | Auth0 OAuth2 client secret | Auth0 Dashboard |
| `SSH_PRIVATE_KEY` | SSH key for server access | Local key generation |
| `SSH_HOST` | Target server hostname | DO Droplet Console |
| `SSH_USER` | SSH username | DO Droplet Console |
| `do-project-ca.crt` | DigitalOcean CA certificate | DO Project Settings |

### Adding New Secrets to GitHub

1. Go to repository **Settings → Secrets and variables → Actions**
2. Add secrets under the `production` environment
3. Required secrets: `SSH_PRIVATE_KEY`, `SSH_HOST`, `SSH_USER`, `DO_DB_HOST`, `DO_DB_PORT`, `DO_DB_NAME`, `DO_DB_USER`, `DO_DB_PASSWORD`, `AUTH0_CLIENT_SECRET`

### Rotating Secrets

1. Update the secret in the source system (DO Console, Auth0 Dashboard)
2. Update the corresponding GitHub Secret
3. Run the **Upgrade n8n** workflow (or Deploy) to apply changes

## Backup

- **Workflows**: Stored in DO Managed PostgreSQL with daily backups
- **Executions**: Daily backup to DO Spaces S3
- **Pre-upgrade**: Automatic `pg_dump` stored at `/opt/loading-n8n/backups/` on the server

### Manual Backup

```bash
docker exec loading-n8n pg_dump \
  -h $DO_DB_HOST -p 25060 -U doadmin -d loading_n8n \
  --no-owner --no-acl > n8n_backup_$(date +%Y%m%d).sql
```

### Restore from Backup

```bash
docker exec -i loading-n8n psql \
  -h $DO_DB_HOST -p 25060 -U doadmin -d loading_n8n \
  < n8n_backup_YYYYMMDD.sql
```

## Ports

| Port | Service | Notes |
|------|---------|-------|
| 5678 | n8n Web UI | Default port |
| 5678 | n8n Healthcheck | `/healthz` endpoint |

## Troubleshooting

### Container Won't Start

Check logs: `docker compose logs n8n`

Common issues:
- Missing environment variables
- Invalid database credentials
- SSL certificate not mounted
- DB migration failure after version upgrade

### Health Check Failing

1. Check `/healthz` endpoint: `curl http://localhost:5678/healthz`
2. View container logs: `docker compose logs --tail 50 n8n`
3. Check if DB migration is in progress (can take several minutes for large DBs)

### Upgrade Failed / Rollback Triggered

1. Check the GitHub Actions run logs for the failure reason
2. The previous version has already been restored automatically
3. Check backup files at `/opt/loading-n8n/backups/` on the server
4. If manual intervention is needed, SSH to the server and verify with `docker compose ps`

### Can't Connect to Database

1. Verify `DO_DB_HOST` is correct
2. Check database credentials in `.env`
3. Ensure VPN/network connectivity to DO database

### SSO Not Working

1. Verify Auth0 callback URL: `https://n8n.loadingtechnology.app/rest/authenticate`
2. Check `N8N_OAUTH2_*` environment variables
3. Validate client ID and secret in Auth0 Dashboard

## License

n8n is open-source under the Sustainable Use License.
