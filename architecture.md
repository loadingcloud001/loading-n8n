# n8n Architecture

## Overview

n8n is an open-source workflow automation platform for connecting services and automating business processes.

## Deployment

```
+-------------------------------------------------------------+
|                  Nginx Reverse Proxy                           |
|                   (auth0-authenticated)                     |
+-------------------------+-----------------------------------+
                          |
                          v
+-------------------------------------------------------------+
|                        n8n                                  |
|                  (Docker Compose)                            |
|                                                               |
|  +---------+  +---------+  +---------+  +---------+          |
|  | Workflow|  |  Node   |  |Webhook  |  |  Exec   |          |
|  | Editor  |  | Library |  | Handler |  |  Log    |          |
|  +---------+  +---------+  +---------+  +---------+          |
+-------------------------+-----------------------------------+
                          |
         +----------------+----------------+
         v                v                v
+-------------+ +-------------------+ +---------+
|    DO      | |       Auth0       | | DO Spaces|
| PostgreSQL | |        (SSO)      | |    S3    |
+-------------+ +-------------------+ +---------+
```

## Data Sources

| Source | Connection | Purpose |
|--------|-----------|---------|
| DO Managed PostgreSQL | PostgreSQL | Workflow and execution data |
| Auth0 | OAuth2 | Single sign-on |
| DO Spaces | S3-compatible | File storage for workflows |

## Ports

| Port | Service | Notes |
|------|---------|-------|
| 5678 | n8n Web UI | Default |

## Auth0 Configuration

| Field | Value |
|-------|-------|
| Application Type | SPA |
| Callback URL | https://n8n.loadingtechnology.app/rest/authenticate |

## Dependencies

```
n8n ------> Auth0 (authentication)
      +----> DO Managed PostgreSQL (workflow storage)
      +----> DO Spaces S3 (file attachments)
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `N8N_HOST` | Hostname |
| `N8N_PORT` | Service port |
| `WEBHOOK_URL` | Public webhook URL |
| `DB_TYPE` | Database type (postgresdb) |
| `DB_POSTGRESDB_HOST` | DO Managed PostgreSQL host |
| `DB_POSTGRESDB_PORT` | PostgreSQL port (25060) |
| `DB_POSTGRESDB_DATABASE` | Database name (`loading_n8n`) |
| `DB_POSTGRESDB_USER` | PostgreSQL user (doadmin) |
| `DB_POSTGRESDB_PASSWORD` | PostgreSQL password |
| `N8N_AUTH_MODE` | Authentication mode (oauth2) |
| `N8N_OAUTH2_CLIENT_ID` | Auth0 client ID |
| `N8N_OAUTH2_CLIENT_SECRET` | Auth0 client secret |
| `N8N_OAUTH2_ISSUER` | Auth0 issuer URL |

## Connection to PostgreSQL

```
DB_POSTGRESDB_HOST=private-loading-postgres-do-user-34964671-0.i.db.ondigitalocean.com
DB_POSTGRESDB_PORT=25060
DB_POSTGRESDB_DATABASE=loading_n8n
DB_POSTGRESDB_USER=doadmin
DB_POSTGRESDB_PASSWORD=<from do003-postgres.age>
DB_POSTGRESDB_SSL=true
```

## Database Architecture

```
DO Managed PostgreSQL — loading-postgres cluster (sgp1, pg16)
├── loading_n8n (11 MB)     ← n8n workflows + stock_news AI rating schema
├── speckle_fresh (58 MB)    ← Speckle BIM/3D data ONLY
├── loading_grafana (7.8 MB) ← Grafana/TimescaleDB metrics
└── defaultdb                ← DO system default
```

Each service has its own dedicated database. No shared tables between services.

## Upgrade Architecture

n8n upgrades are managed through GitHub Actions with automated safety mechanisms.

### Upgrade Flow

```
GitHub Actions (upgrade.yml)
  |
  ├─ 1. Capture current version (for rollback)
  ├─ 2. Pre-upgrade DB backup (pg_dump)
  ├─ 3. Pull target image: n8nio/n8n:${N8N_VERSION}
  ├─ 4. Graceful shutdown (30s stop_grace_period)
  ├─ 5. Start new version (auto-migration runs)
  ├─ 6. Health check with retry (up to 10 attempts, 6s apart)
  │     ├─ PASS → Verify & complete
  │     └─ FAIL → Rollback to previous version
  └─ 7. Final status report
```

### Version Management

| Component | Detail |
|-----------|--------|
| Image tag | `n8nio/n8n:${N8N_VERSION:-latest}` |
| Version control | `.env` file or workflow input |
| Rollback | Automated restore to previous version on health failure |
| Backups | Auto-created before upgrade, kept for 10 most recent |

### Key Safety Features

1. **Pre-upgrade database backup** — Full `pg_dump` before any version change
2. **Graceful shutdown** — 30-second `stop_grace_period` for in-flight executions
3. **Health check with retry** — Up to 10 attempts with 6s interval using `/healthz`
4. **Automated rollback** — If health check fails, previous version is restarted
5. **Version pinning** — No more `latest` by default; explicit version targeting
6. **Data persistence** — Named Docker volume `loading-n8n-data` survives upgrades

### Workflow Triggers

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `ci.yml` | Push/PR to `main` | Validate compose syntax, check for leaked secrets |
| `deploy.yml` | Push to `main` or manual | Standard deployment with optional backup |
| `upgrade.yml` | Manual dispatch | Versioned upgrade with full safety flow |

## Backup

- Workflows: Stored in DO Managed PostgreSQL
- Executions: Daily backup to DO Spaces
- Pre-upgrade: Automatic `pg_dump` stored at `/opt/loading-n8n/backups/`