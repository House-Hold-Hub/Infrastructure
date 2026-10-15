# Infrastructure

Local runtime configuration for House Hold Hub (ownership per ADR-009: Infrastructure owns
local full-stack orchestration, credentials and secret rotation, consuming the service-owned
images of Backend and Frontend).

This repository currently defines the **environment and secret contract** (Infrastructure#2).
The Docker Compose definition belongs to Infrastructure#1 and is **not** included here.

## Files

| File | Purpose |
|---|---|
| `.env.example` | Documented template: purpose, example and requiredness of every variable |
| `.gitignore` | Blocks `.env` and `.env.*` (except `.env.example`) from being committed |

## Configure

```bash
cp .env.example .env
```

Then edit `.env`:

1. Replace `POSTGRES_PASSWORD` and `DJANGO_SECRET_KEY` with values you generate locally
   (`openssl rand -base64 24` / `openssl rand -base64 48`). The placeholders are not secrets
   and must never be reused outside your machine.
2. Leave the rest at the local defaults unless you know you need to change them.

Variables are grouped in `.env.example` by consumer:

- **PostgreSQL** (`POSTGRES_*`): read by the Backend and, by the same names, by the official
  PostgreSQL image. PostgreSQL 14+ is the approved family; the exact version is set by Infrastructure#1.
- **Backend runtime** (`DJANGO_*`, `WEB_CONCURRENCY`): read at process start from `config/settings.py`
  and the Backend `Dockerfile`. `DJANGO_SECRET_KEY` has no default and the Backend will not start without it.
- **Frontend build-time** (`VITE_API_BASE_URL`): embedded in the JavaScript bundle by Vite at build
  time. It is **public**; change it by rebuilding, never put secrets in any `VITE_*` variable.

The variable list was checked against Backend `config/settings.py` + `Dockerfile` and Frontend
`src/config.ts` + `Dockerfile` (as of Backend `3d4d7b2`, Frontend `5c9d122`). When a service adds
or removes a variable, update `.env.example` in the same change set.

## Secrets: do not commit `.env`

> **Never commit `.env` or any file containing credentials.** `.gitignore` blocks `.env`, `.env.*`
> and `*.env` (keeping only `.env.example`), but it does not protect files that are already tracked.
> If a secret is committed by mistake, rotate it immediately; deleting the commit is not enough.

- `.env.example` contains placeholders only. Keep it free of real values.
- Do not paste secrets into issues, PRs, logs or chat.
- Do not use real personal data locally. **D03** (launch jurisdiction and retention) must be
  resolved before any real personal data or real-email staging data is processed; use synthetic data.

## Deferred decisions (no provider is selected here)

The contract is provider-neutral. It deliberately contains no email, deployment, secret-store or
monitoring variables.

- **D02**: the transactional email provider is selected before M1 email integration (ADR-015, behind
  the Backend's provider-neutral adapter); deployment, secret-store and monitoring providers are
  selected before M9. Variables for them will be added only after those decisions.
- **D03**: gates real personal data, including real-email staging.

## Run locally

The full stack (PostgreSQL + Backend + Frontend) is orchestrated by Docker Compose, owned by
Infrastructure#1, which is **not yet available**. Until then, the full-stack startup from
`.env.example` has **not been tested**; do not treat this contract as startup-verified.

What can be done today, with services run from their own repositories:

```bash
# PostgreSQL: any local PostgreSQL 14+ using the POSTGRES_* values from .env
set -a; source .env; set +a

# Backend (from the Backend repository)
uv sync && uv run python manage.py migrate && uv run python manage.py runserver

# Frontend (from the Frontend repository): VITE_API_BASE_URL is read at build/dev time
npm ci && npm run dev
```

### Known points for Infrastructure#1 to settle

- Backend documents a same-origin default topology without CORS, while the Frontend defaults
  `VITE_API_BASE_URL` to `http://localhost:8000` (cross-origin from the dev server). The Compose
  work must either serve both behind one origin or have the cross-origin case reviewed.
- `POSTGRES_HOST` must be the Compose service name when the Backend runs in a container.
