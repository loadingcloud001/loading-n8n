# n8n Architecture

## Overview

n8n is an open-source workflow automation platform for connecting services and automating business processes.

## Deployment

```
┌─────────────────────────────────────────────────────────┐
│                  Nginx Reverse Proxy                     │
│                   (auth0-authenticated)                 │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                        n8n                              │
│                  (Docker Compose)                        │
│                                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │ Workflow│  │  Node   │  │Webhook  │  │  Exec   │  │
│  │ Editor  │  │ Library │  │ Handler │  │  Log    │  │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘  │
└─────────────────────┬───────────────────────────────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
┌─────────────┐ ┌───────────┐ ┌─────────┐
│ loading-    │ │   Auth0   │ │ Backblaze│
│ postgres    │ │  (SSO)   │ │   B2    │
└─────────────┘ └───────────┘ └─────────┘
```

## Data Sources

| Source | Connection | Purpose |
|--------|-----------|---------|
| loading-postgres | PostgreSQL | Workflow and execution data |
| Auth0 | OAuth2 | Single sign-on |
| Backblaze B2 | S3-compatible | File storage for workflows |

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
n8n ─────────→ Auth0 (authentication)
      └────────→ loading-postgres (workflow storage)
      └────────→ Backblaze B2 (file attachments)
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `N8N_HOST` | Hostname |
| `N8N_PORT` | Service port |
| `WEBHOOK_URL` | Public webhook URL |
| `DB_TYPE` | Database type (postgresdb) |
| `DB_POSTGRESDB_HOST` | loading-postgres |
| `DB_POSTGRESDB_PORT` | PostgreSQL port (5432) |
| `DB_POSTGRESDB_DATABASE` | Database name |
| `DB_POSTGRESDB_USER` | PostgreSQL user |
| `DB_POSTGRESDB_PASSWORD` | PostgreSQL password |
| `N8N_AUTH_MODE` | Authentication mode (oauth2) |
| `N8N_OAUTH2_CLIENT_ID` | Auth0 client ID |
| `N8N_OAUTH2_CLIENT_SECRET` | Auth0 client secret |
| `N8N_OAUTH2_ISSUER` | Auth0 issuer URL |

## Connection to PostgreSQL

```
DB_POSTGRESDB_HOST=loading-postgres
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=postgres
DB_POSTGRESDB_USER=postgres
DB_POSTGRESDB_PASSWORD=<password>
```

## Backup

- Workflows: Stored in loading-postgres
- Executions: Daily backup to Backblaze B2
