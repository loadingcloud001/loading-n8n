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

## Configuration

### Environment Variables

Copy `.env.example` to `.env` and configure:

```bash
cp .env.example .env
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
DO_DB_HOST=private-speckle-postgres-do-user-34964671-0.h.db.ondigitalocean.com
DO_DB_PORT=25060
DO_DB_NAME=speckle_fresh
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

- Docker and Docker Compose installed
- Access to `nginx-network` external network
- DO Managed PostgreSQL instance
- Auth0 application configured

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
| `do-project-ca.crt` | DigitalOcean CA certificate | DO Project Settings |

### Rotating Secrets

1. Update the secret in your secrets manager (DO Spaces, Vault, etc.)
2. Restart the n8n container: `docker compose restart n8n`

## Backup

- **Workflows**: Stored in DO Managed PostgreSQL with daily backups
- **Executions**: Daily backup to DO Spaces S3

## Ports

| Port | Service | Notes |
|------|---------|-------|
| 5678 | n8n Web UI | Default port |

## Troubleshooting

### Container Won't Start

Check logs: `docker compose logs n8n`

Common issues:
- Missing environment variables
- Invalid database credentials
- SSL certificate not mounted

### Can't Connect to Database

1. Verify `DO_DB_HOST` is correct
2. Check database credentials in `.env`
3. Ensure VPN/network connectivity to DO database

### SSO Not Working

1. Verify Auth0 callback URL: `https://n8n.loadingtechnology.app/rest/authenticate`
2. Check `N8N_OAUTH2_*` environment variables
3. Validate client ID and secret in Auth0 Dashboard

## License

n8n is open-source under the Apache 2.0 license.
