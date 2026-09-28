# HouseHoldHub Infrastructure

Infrastructure-owned runtime configuration for HouseHoldHub.

This repository owns deployment manifests, runtime-target configuration, managed-service provisioning, credentials/secret rotation, and provider health configuration. Service Dockerfiles remain owned by Backend and Frontend.

## Environment contract

Copy the committed example before local use:

```bash
cp .env.example .env
```

**Never commit `.env`, real credentials, production secrets, or provider tokens.** The repository `.gitignore` excludes local environment files while keeping `.env.example` versioned.

The current contract mirrors the variables consumed by the existing Backend and Frontend M0 runtimes:

| Variable | Requiredness | Purpose | Local example |
|---|---|---|---|
| `DJANGO_SECRET_KEY` | Required by this local contract | Django cryptographic signing secret | `local-development-only-change-me` |
| `DJANGO_DEBUG` | Required by this local contract | Enables local Django debug behavior | `true` |
| `DJANGO_ALLOWED_HOSTS` | Required by this local contract | Comma-separated hosts accepted by Django | `localhost,127.0.0.1` |
| `POSTGRES_DB` | Required by this local contract | PostgreSQL database name | `householdhub` |
| `POSTGRES_USER` | Required by this local contract | PostgreSQL application user | `householdhub` |
| `POSTGRES_PASSWORD` | Required by this local contract | PostgreSQL application password | `householdhub` (local only) |
| `POSTGRES_HOST` | Required by this local contract | PostgreSQL host reachable by Backend | `localhost` |
| `POSTGRES_PORT` | Required by this local contract | PostgreSQL TCP port | `5432` |
| `VITE_API_BASE_URL` | Required by this local contract | Vite development/build API base path | `/api/v1` |

All values in `.env.example` are development-safe examples. A copied `.env` therefore needs no provider credential or unresolved placeholder for the current M0 service configuration.

The Backend currently reads the Django/PostgreSQL variables above. The Frontend consumes `VITE_API_BASE_URL` during Vite development/build. Infrastructure issue #1 owns the full-stack Docker Compose definition and will consume this contract; this issue intentionally does not add orchestration or service-owned Dockerfiles.

## Deferred provider configuration

The contract is intentionally provider-neutral. Do not introduce vendor-specific variable names merely to prepare for a future provider.

- **D02 — email:** select the managed transactional-email provider before M1 email integration.
- **D02 — production runtime:** select deployment, secret-store, and monitoring providers before M9 provisioning/launch.
- **D03 — real data:** resolve launch jurisdiction and retention requirements before processing real personal data, including real-email staging.

After D02 resolves a provider, extend `.env.example` with only the variables actually required by that integration, documenting purpose, example, and requiredness here. Real credentials must be injected through the selected secret mechanism and must never be committed.

Transactional email remains behind the Backend-owned provider-neutral adapter. Infrastructure owns provider provisioning, verified sender/domain configuration, SPF/DKIM/DMARC, credentials, rotation, and provider health. No queue, Redis, Celery, or dedicated worker is introduced solely for MVP email.

## Canonical references

- [Product roadmap — deferred decisions D01–D06](https://github.com/House-Hold-Hub/Documentation/blob/main/product/roadmap.md)
- [Security model](https://github.com/House-Hold-Hub/Documentation/blob/main/security/security-model.md)
- [ADR-015: Transactional email delivery](https://github.com/House-Hold-Hub/Documentation/blob/main/architecture/adr/ADR-015-transactional-email-delivery.md)
- [Technology baseline](https://github.com/House-Hold-Hub/Documentation/blob/main/architecture/technology-baseline.md)
- [ADR-009: Five-repository topology](https://github.com/House-Hold-Hub/Documentation/blob/main/architecture/adr/ADR-009-five-repository-topology.md)
