# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Infrastructure is one of the five HouseHoldHub repositories (Backend, Frontend, Infrastructure,
Automation, Documentation: ADR-009). It owns local full-stack orchestration (Docker Compose),
deployment manifests, runtime-target configuration, managed-service provisioning, credentials and
secret rotation, provider health configuration, and the exact deployed PostgreSQL (14+ family) and
service versions. It consumes the images Backend and Frontend build from their own Dockerfiles:
never copy or fork a service Dockerfile or workflow here. Reusable CI workflows belong to
Automation, the API contract and the decisions to Documentation (`../Documentation` when cloned
alongside: ADR-009, `architecture/technology-baseline.md`, `security/security-model.md`, deferred
decisions D01–D06 in `product/roadmap.md`). Live GitHub issues, not Markdown, are authoritative for
execution state.

Check the tree before describing what exists. The environment and secret contract (`.env.example`,
`.gitignore`, README) is the core of the repository. `compose.yaml` and `nginx/default.conf` define the local stack (README "Run locally", with the dated
verification record); never describe startup as working beyond what that record says, and re-run
`docker compose up -d --build` after changing either. Services can also run from their own repository
with `.env` exported (`set -a; source .env; set +a`).

## Commands

Beyond the `Secret scan / scan` gate there is no build, lint or test tooling unless `.github/workflows` shows
otherwise. Sibling paths
assume the main checkout with the other repositories cloned alongside; they do not resolve from a
`.claude/worktrees/*` worktree.

```bash
# Local setup, then replace POSTGRES_PASSWORD (openssl rand -base64 24) and DJANGO_SECRET_KEY (openssl rand -base64 48)
cp .env.example .env

# Contract drift: variables the services read on their main vs those .env.example defines (commented ones included).
# No output = in sync; "<" read by a service but missing here; ">" defined here but read by no service.
git -C ../Backend fetch -q && git -C ../Frontend fetch -q
diff <(
  { git -C ../Backend show origin/main:config/settings.py | grep -oE 'env(_[a-z]+)?\("[A-Z0-9_]+"' | cut -d'"' -f2
    git -C ../Frontend grep -h -oE 'import\.meta\.env\.VITE_[A-Z0-9_]+' origin/main -- src | sed 's/.*\.//'
    for r in Backend Frontend; do git -C ../$r show origin/main:Dockerfile; done |
      grep -oE '^ARG [A-Z0-9_]+|\$\{[A-Z0-9_]+' | sed -E 's/^ARG //; s/^\$\{//'
  } | sort -u
) <(grep -oE '^(# )?[A-Z][A-Z0-9_]*=' .env.example | sed -E 's/^# //; s/=$//' | sort -u)

# Secret scan as CI runs it on a pull request, with the Gitleaks image read through the Automation pin in gates.yml
sha=$(grep -oE 'secret-scan\.yml@[0-9a-f]{40}' .github/workflows/gates.yml | cut -d@ -f2)
image=$(git -C ../Automation show "${sha:?no Automation pin in gates.yml}:.github/workflows/secret-scan.yml" \
  | grep -oE 'ghcr\.io/gitleaks/gitleaks@sha256:[0-9a-f]{64}')
docker run --rm -v "$PWD:/repo:ro" -e GIT_CONFIG_COUNT=1 -e GIT_CONFIG_KEY_0=safe.directory -e GIT_CONFIG_VALUE_0=/repo \
  "$image" git /repo --log-opts "origin/main..HEAD" --redact --verbose --no-banner
```

The drift patterns match how the services read variables today (Backend `env*()` helpers,
Frontend `import.meta.env.VITE_*`, Dockerfile `ARG` and `${…}`); extend them if that changes.

Gitleaks exits 0 even when git fails inside the container (`ERR [git] fatal: …` followed by
`0 commits scanned`, `no leaks found`). This happens from a worktree, whose `.git` file points outside
the mount, and intermittently right after a commit. Trust a run only if `N commits scanned` equals
`git rev-list --count --no-merges origin/main..HEAD`; for a worktree's branch, run from the main
checkout with `origin/main..<branch>`. Without `--log-opts` it scans the full history, as the push
to `main` does.

## The environment contract

`.env.example` mirrors what the services read, nothing more: Backend `config/settings.py` and
`Dockerfile` (`WEB_CONCURRENCY` in the gunicorn CMD, ignored by `runserver`), Frontend
`src/config.ts` and `Dockerfile`. When a service adds or removes a variable, change `.env.example`
in the same change set and update the Backend/Frontend commits the README says it was checked
against.

- Grouped by consumer: `POSTGRES_*` (the Backend, and by the same names the official PostgreSQL
  image), Backend runtime `DJANGO_*` and `WEB_CONCURRENCY` (read at process start), Frontend
  build-time `VITE_*` (baked into the public bundle, so never a secret; a change needs a rebuild).
- Each entry is tagged `[required]` or `[optional, default: X]` with X the service's own code
  default, plus `SECRET` where it applies; the assigned value is the local one (`DJANGO_DEBUG=true`
  against a default of false). Optional variables whose default already suits local runs stay
  commented out. `POSTGRES_PASSWORD` is required for the stack although the Backend defaults it to
  empty, because the PostgreSQL image refuses to start without one.
- Provider-neutral on purpose: no email, deployment, secret-store or monitoring variables or vendor
  configuration until D02 selects those providers (email before M1 integration, the rest before M9
  provisioning). D03 gates any real personal data, including real-email staging: synthetic data only.
- Placeholders stay visibly fake (`change-me-local-only…`). Gitleaks' default `generic-api-key`
  rule flags realistic values for `POSTGRES_PASSWORD` or `DJANGO_SECRET_KEY`, and the pull-request
  scan covers every commit the PR adds, so removing a value in a later commit does not clear it:
  rewrite the commit, and rotate anything that was pushed.
- `.gitignore` ignores `.env`, `.env.*`, `*.env` and `.envrc`, so a tracked template must end in
  `.env.example`; it cannot untrack a file that is already committed.

## Constraints for the Compose work

- Same-origin is implemented by mounting `nginx/default.conf` over the Frontend image's site (no extra
  service, no copied Dockerfile); Compose overrides `POSTGRES_HOST=db` and keeps `VITE_API_BASE_URL` empty.
- The Backend assumes a same-origin topology and configures no CORS, and the Frontend client accepts only an
  empty `VITE_API_BASE_URL` or the page's own origin. Serve both behind one origin, and have any
  cross-origin case reviewed first; check the Frontend image's `VITE_API_BASE_URL` default against this.
- `POSTGRES_HOST` must be the Compose service name when the Backend runs in a container.

## CI gate and branch protection

- `.github/workflows/gates.yml` only calls Automation's reusable `secret-scan.yml`, pinned by full
  commit SHA, never a branch or tag. Nothing bumps the pin: change it by hand together with the SHA
  the README quotes (Automation's `docs/shared-gates.md` records the current shared pin).
- `main` requires the published check `Secret scan / scan` (caller job name / called job name):
  renaming the job silently drops the protection, and a `paths:` filter would leave the required
  check pending. Keep `permissions: contents: read` and no secrets.
- There is no `.gitleaks.toml` or allowlist; adding one is a deliberate decision because this
  repository owns the secret contract.
- Require a new check only once a workflow publishes it (confirm the name from a PR run:
  `gh api repos/House-Hold-Hub/Infrastructure/commits/<sha>/check-runs -q '.check_runs[].name'`),
  and apply it with Automation's `scripts/set-required-checks.sh` (admin rights). A dependency scan
  waits for a lockfile; OpenAPI validation does not apply here.
- Protection also requires a pull request, applies to admins, and forbids force-push and deletion.
  GitHub-native secret scanning, push protection and Dependabot security updates are administrator
  settings, so do not assume they are on: a secret pushed to an unprotected branch is caught only once a
  PR is opened. The README records the live state as dated facts: re-read it with `gh api` before
  editing them.

## Git workflow

Branches are `<type>/<issue>-<slug>`, prefixed with the repository for another repository's issue
(`chore/2-env-secret-contract`, `chore/automation-2-shared-gates`); throwaway PRs proving the gate
blocks use `test/…` and are closed unmerged. Conventional Commit subjects, `Refs #<n>` (or
`Refs House-Hold-Hub/<Repo>#<n>`) in the body, pull requests into `main` that state what was
verified and what was not.
