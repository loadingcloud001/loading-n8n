# loading-n8n

## Service Purpose

n8n is an open-source workflow automation platform for connecting services and automating business processes.

## Environment

- Deployment: Docker Compose
- Default image: `n8nio/n8n`
- Port: `5678` (default)
- Auth: Auth0 (SSO)

## Configuration

### Environment Variables

| Variable | Purpose | Value |
|----------|---------|-------|
| `N8N_HOST` | Hostname | `n8n.loadingtechnology.app` |
| `N8N_PORT` | Port | `5678` |
| `WEBHOOK_URL` | Webhook URL | `https://n8n.loadingtechnology.app` |
| `DB_TYPE` | Database type | `postgresdb` |
| `DB_POSTGRESDB_HOST` | PostgreSQL host | `cmp-postgres-do-user-31385694-0.e.db.ondigitalocean.com` |
| `DB_POSTGRESDB_PORT` | PostgreSQL port | `25060` |
| `DB_POSTGRESDB_DATABASE` | Database name | `loading_postgres` |
| `DB_POSTGRESDB_USER` | User | `doadmin` |
| `DB_POSTGRESDB_PASSWORD` | Password | From DO Managed PostgreSQL |
| `DB_POSTGRESDB_SSL` | SSL required | `true` |
| `DB_POSTGRESDB_SSL_REJECT_UNAUTHORIZED` | SSL verification | `false` |
| `N8N_AUTH_MODE` | Auth mode | `oauth2` |
| `N8N_OAUTH2_CLIENT_ID` | OAuth2 Client ID | `ws8wBaPdXYFWwNAtva4YWQmCuT1Ah0AT` |
| `N8N_OAUTH2_CLIENT_SECRET` | OAuth2 Secret | From `auth0.age` secrets |
| `N8N_OAUTH2_ISSUER` | OAuth2 Issuer | `https://dev-63h751qzectzner7.us.auth0.com/` |

### Auth0 Configuration

- **Tenant Domain:** `dev-63h751qzectzner7.us.auth0.com`
- **Client ID:** `ws8wBaPdXYFWwNAtva4YWQmCuT1Ah0AT`
- **Callback URL:** `https://n8n.loadingtechnology.app/rest/authenticate`

## Dependencies

### Services Used
- Auth0 (SSO login at `dev-63h751qzectzner7.us.auth0.com`)
- DO Managed PostgreSQL (PostgreSQL at `cmp-postgres-do-user-31385694-0.e.db.ondigitalocean.com:25060`)

### Services That Depend On It
- Other services needing workflow automation

## Resources

- Dev-Standards: https://github.com/loadingcloud001/dev-standards/blob/main/RESOURCES.md
- n8n Docs: https://docs.n8n.io/

## Secrets

Encrypted in dev-standards `.secrets/`:
- `auth0.age` - SSO credentials

## Deployment Status

**Deployed:** Yes - `loading-n8n` container running on droplet `157.245.148.63`

### Droplet Deployment Path
```
/opt/loading-n8n/
├── docker-compose.yml
└── .env (contains POSTGRES_PASSWORD, DB_POSTGRESDB_PASSWORD, AUTH0_CLIENT_SECRET)
```

### Quick Deploy Commands
```bash
# SSH to droplet
ssh root@157.245.148.63

# Restart n8n
cd /opt/loading-n8n && docker-compose restart

# View logs
docker logs loading-n8n

# Verify
curl http://localhost:5678/
```

## Deployment Checklist

- [x] Configure Auth0 Application
- [x] Connect to DO Managed PostgreSQL
- [x] Set up OAuth2 authentication
- [x] Deploy n8n container
- [x] Verify: `curl http://localhost:5678/` returns HTTP 200
- [x] Test SSO login at `https://n8n.loadingtechnology.app`
- [ ] Create first workflow
