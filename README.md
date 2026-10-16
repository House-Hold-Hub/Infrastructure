# Infrastructure

Local runtime configuration for House Hold Hub (ownership per ADR-009: Infrastructure owns
local full-stack orchestration, credentials and secret rotation, consuming the service-owned
images of Backend and Frontend).

This repository defines the **environment and secret contract** (Infrastructure#2) and the **local
Docker Compose stack** (Infrastructure#1): PostgreSQL, Backend and Frontend behind one origin.

## Files

| File | Purpose |
|---|---|
| `.env.example` | Documented template: purpose, example and requiredness of every variable |
| `compose.yaml` | Local stack: `db`, one-shot `migrate`, `backend`, `frontend` |
| `nginx/default.conf` | Same-origin edge: SPA plus reverse proxy of `/api/` and `/admin/` to the Backend |
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
  PostgreSQL image. PostgreSQL 14+ is the approved family; the local stack pins `postgres:17-alpine` in `compose.yaml`.
- **Backend runtime** (`DJANGO_*`, `WEB_CONCURRENCY`): read at process start from `config/settings.py`
  and the Backend `Dockerfile`. `DJANGO_SECRET_KEY` has no default and the Backend will not start without it.
- **Frontend build-time** (`VITE_API_BASE_URL`, empty for same-origin): embedded in the JavaScript bundle by Vite at build
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

## Run locally (Docker Compose)

Prerequisites: Docker Engine with Compose v2 (verified with Docker 29.8 / Compose 5.6), and the
Backend, Frontend and Infrastructure repositories cloned as siblings (`../Backend`, `../Frontend`).
Run from the main checkout of Infrastructure, not from a `.claude/worktrees/*` worktree. The service
Dockerfiles are built from the sibling repositories and are not copied here.

```bash
cp .env.example .env     # then set POSTGRES_PASSWORD and DJANGO_SECRET_KEY (see Configure)
docker compose up -d --build
```

Startup order is enforced: `db` (healthcheck `pg_isready`) -> `migrate` (one-shot
`manage.py migrate --noinput`, must exit 0) -> `backend` (healthcheck) -> `frontend`. The app is at
<http://localhost:8080> (published on 127.0.0.1 only; PostgreSQL publishes no port).

- **Same origin, no CORS.** The Frontend container's nginx (`nginx/default.conf`, mounted over the
  image's default site) serves the SPA and proxies `/api/` and `/admin/` to `backend:8000`, keeping the
  `Host` header so sessions and CSRF work. `VITE_API_BASE_URL` must stay empty (relative URLs); the
  client rejects any other cross-origin base. The Django admin is served without static assets
  (the Backend image does not collect them), so it is unstyled.
- **Database connectivity.** Compose sets `POSTGRES_HOST=db` and `POSTGRES_PORT=5432` for the Backend
  and migration job whatever `.env` says (those values in `.env` are for services run on the host).
- **Cookies.** `DJANGO_SECURE_COOKIES` defaults to `false` in Compose because the local stack is plain
  HTTP. Use another host name only after adding it to `DJANGO_ALLOWED_HOSTS`.
- **Migrations.** They run on every `up` through the `migrate` service and are idempotent
  ("No migrations to apply" on later runs). Re-run manually: `docker compose run --rm migrate`.
- **Persistence.** Data lives in the named volume `householdhub_pgdata`; it survives `down` and restarts.
- **`POSTGRES_*` initialise only an empty database.** `POSTGRES_DB`, `POSTGRES_USER` and
  `POSTGRES_PASSWORD` are read by the PostgreSQL image only when `pgdata` is empty (first start). Afterwards
  the role, database and password live inside the volume, while the Backend and `migrate` read `.env` on every
  `up`. Editing these values in `.env` alone therefore makes the Backend fail to authenticate
  (see "Rotate the PostgreSQL password" below).
- **Changing `VITE_*`** needs `docker compose build frontend` (the value is baked into the bundle).

### Rotate the PostgreSQL password (keeps data)

Rotation changes the credential stored in the existing volume and then `.env`, in this order, so data is
preserved. Do not edit `.env` first.

```bash
# 1. With the stack running and the OLD .env still in place, set the new password inside PostgreSQL.
#    \password prompts twice and sends the hash, so the password stays out of shell history and logs.
docker compose exec db sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
#    postgres=> \password householdhub      (use your POSTGRES_USER), then \q

# 2. Put the same value in .env as POSTGRES_PASSWORD.
# 3. Recreate the services so they pick it up (volume and data are untouched).
docker compose up -d
```

If `.env` was already edited first, the Backend cannot connect: restore the old value in `.env`, run
`docker compose up -d`, and follow the steps above. Rotation is not the same as
`docker compose down -v`, which **deletes the volume and all data** and is only a reinitialisation:
the next `up` creates a fresh empty database from whatever `.env` says. Use it only when losing the local
data is acceptable. Renaming `POSTGRES_DB` or `POSTGRES_USER` on an existing volume is not covered by
rotation; it needs SQL (`ALTER DATABASE/ROLE ... RENAME`) or a reinitialisation.

Stop the stack:

```bash
docker compose down        # keeps the database volume
docker compose down -v     # DESTRUCTIVE: deletes the database volume (all local data); not a credential rotation
```

Useful: `docker compose ps -a`, `docker compose logs -f backend`, `docker compose config -q` (validates
the file and the required `.env` values).

### Verification (2026-10-16, Docker 29.8.2, Compose 5.6.0, Backend/Frontend `origin/main`)

Run: `config -q`, full `up -d --build`, `migrate` exited 0 and applied all migrations, `GET /` -> 200
(SPA), `GET /api/v1/` -> Backend JSON 404 through the proxy, `/admin/login/` -> 200, a same-origin
POST with the CSRF cookie/token passed CSRF (200) while one without token got 403, the built bundle
contains no `localhost:8000`, a row written to PostgreSQL survived `down` + `up` and the second
`migrate` applied nothing.

Credential rotation (2026-10-16): with data in the volume, editing `.env` first made `migrate` fail with
"password authentication failed"; restoring the old value, changing the role password in PostgreSQL
(`ALTER ROLE`, the statement `\password` issues) and then updating `.env` + `docker compose up -d` restored a healthy
stack with the data intact. The interactive `\password` prompt itself was not driven.

Not verified: a browser login/registration flow end to end (no verified flow was exercised; only
HTTP-level checks), TLS or non-localhost hosts, `DJANGO_DEBUG=true` runs, and Apple-silicon vs amd64 differences.

### Running services without Compose

```bash
set -a; source .env; set +a          # POSTGRES_HOST=localhost: needs your own PostgreSQL 14+
uv sync && uv run python manage.py migrate && uv run python manage.py runserver   # in Backend
npm ci && npm run dev                                                              # in Frontend
```

The Vite dev server is cross-origin to the Backend (`VITE_API_BASE_URL` empty would hit the Vite
server itself); that case needs its own review and is not supported by the Compose stack.

## CI gate and branch protection

`.github/workflows/gates.yml` (pull requests and pushes to `main`, no path filters,
`permissions: contents: read`, no secrets) calls the shared Gitleaks workflow from
[Automation#2](https://github.com/House-Hold-Hub/Automation/issues/2), pinned to the Automation
`main` commit `5eeae49dfbef5c30fadb4d2ab8a83db068c01488`. See Automation `docs/shared-gates.md`.

| Item | Value |
|---|---|
| Published check name | `Secret scan / scan` |
| Scope on pull requests | Commits the PR adds (findings are redacted) |
| Scope on push to `main` | Whole git history |
| Allowlists | None. `.gitleaks.toml` is absent; adding one is a deliberate decision, because this repository owns the secret contract |

The whole history was scanned with the same digest-pinned Gitleaks image before adoption: no findings.

`main` protection (applied and verified on GitHub on 2026-10-16): pull request required
(0 approvals), required status check `Secret scan / scan` bound to GitHub Actions (app 15368),
enforced for administrators, no force-push, no deletion, "require branches to be up to date"
off. Applied with Automation's `scripts/set-required-checks.sh` at the
SHA above. No other check is required because no other workflow exists; dependency scanning and
OpenAPI validation do not apply (no lockfile, no specification). Add checks only when a workflow
that publishes them exists.

### Limits: GitHub-native security features

These are repository settings, not required checks, and this repository's code cannot enable or
verify them. Checked through the API on 2026-10-16, all are **disabled**: secret scanning, push
protection (and secret scanning non-provider patterns and validity checks), and Dependabot
security updates. Enabling them is an administrator decision outside this change. Until then
Gitleaks on pull requests is the only automated secret detection, and a secret pushed directly to a
non-protected branch is detected only when a pull request is opened. Pinned action SHAs are not
auto-updated.
