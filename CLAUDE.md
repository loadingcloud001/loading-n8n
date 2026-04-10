# loading-n8n

Workflow automation platform for connecting services.

## Deploy

**MANDATORY: Use GitHub Actions CI/CD only. Never manual SSH deploy.**

- SSH only for: debugging, reading logs, health checks
- Resource limits: 1G memory, 0.5 CPU
- Log rotation: max-size 10m, max-file 3

## Config

- Port: 5678
- URL: https://n8n.loadingtechnology.app
- Auth: Auth0 SSO
- Database: DO Managed PostgreSQL

## Dependencies

- Auth0 (SSO)
- DO Managed PostgreSQL

## Secrets

Encrypted in dev-standards `.secrets/auth0.age`

## Resources

- [Dev-Standards](https://github.com/loadingcloud001/dev-standards)
- [n8n Docs](https://docs.n8n.io/)
