This file is symlinked and shared across multiple projects; always treat it as a symlink.

# AGENTS.md — Django Backend (uv + DRF + Postgres + Mongo + Huey)

Purpose: This guide tells agents how to work in any Django backend following our baseline patterns. Be concise, opinionated, and ask clarifying questions first.

## Startup Checklist (Agent)
- Read this file fully.
- Use uv with `pyproject.toml` and `uv.lock` (pip/requirements are not used).
- Ensure a local `.env` exists at the project root (values provided via your secret manager or a template).
- Ask: “What should we work on today?” Then propose a short plan and confirm.

## Commands

- Environment
  - Create venv + install all groups: `uv venv && uv sync --all-groups`
  - `.env` at the project root is read via python‑dotenv; keep it present for all runs
- Run
  - Dev server: `uv run python manage.py runserver <port>`
  - Migrations: `uv run python manage.py makemigrations && uv run python manage.py migrate`
- Tests
  - Django tests: `uv run python manage.py test`
  - Coverage (all suites): `uv run coverage run manage.py test`
  - Coverage report: `uv run coverage report -m` and `uv run coverage html`
- Lint
  - Ruff (lint + fix): `uv run ruff check . --fix`
- Background Tasks
  - Huey worker: `uv run python manage.py run_huey --workers <N>`
  - Start workers only on production servers; do not run workers locally.
- Introspection
  - GraphQL schema dump (if present): `uv run python manage.py graphql_schema --schema <module>.schema --out schema.(json|graphql)`

---

## Tech Context (Baseline)
- Django with Django REST Framework for REST APIs.
- PostgreSQL as the primary datastore.
- MongoDB for logs/documents or large, non‑relational payloads (via `pymongo`).
- Settings from `.env` (loaded by python‑dotenv).
- ASGI served in production via Gunicorn with Uvicorn worker.
- Background jobs with Huey; started via `manage.py run_huey`.
- API versioning via DRF namespace/versioning.
 - GraphQL may exist historically; it is being deprecated. Do not add new GraphQL. Prefer simple REST views and migrate GraphQL features to REST when touching related areas.

---

## Project Structure (Baseline)
```
project_root/
  manage.py                    # Django entrypoint
  <project_module>/            # settings.py, urls.py, asgi.py, wsgi.py
  <app_one>/                   # Django app (domain)
    migrations/
    api/<vN>/                  # REST API modules (DRF) when present
    graphql/                   # Legacy GraphQL modules (being deprecated)
    management/commands/       # custom manage.py commands
    templates/                 # app templates (e.g., emails)
    tasks.py                   # background tasks (Huey) when present
    tests/                     # tests colocated (when used)
  <app_two>/
  portal/                      # optional embedded client app (built output checked in)
  http_client/                 # IDE HTTP client requests + env templates
  .infra/                      # production configs (systemd, nginx) and helper scripts
  .env                         # runtime configuration (python-dotenv)
  .template.env                # template for .env values
  pyproject.toml
  uv.lock
```
Conventions:
- Keep views thin; put business logic in service modules or helpers inside each app.
- DRF: structure REST APIs under `api/<version>/` within each app; define serializers for all inputs/outputs.
- Embedded client: if a static client build is included (e.g., under `portal/`), add its build paths to Django `TEMPLATES` and `STATICFILES_DIRS`.
- Avoid cross‑DB transactions; treat Postgres and Mongo as separate boundaries.

---

## Patterns & Practices
- REST‑first: implement simple REST views with DRF (`APIView`, generics, or viewsets). Avoid adding GraphQL; migrate legacy GraphQL endpoints to REST incrementally.
- Views: thin controllers. Heavy work in services/helpers; background work in Huey tasks.
- DRF: explicit serializers and viewsets/generics; paginate lists; validate query params.
- Transactions: use atomic blocks for relational changes; do not mix Postgres invariants with Mongo writes.
- Settings: all dynamic values via environment variables; keep a template with all required keys.
- Security: strict CORS allowlists; protect unsafe methods with CSRF where applicable; never log secrets/PII.
- ASGI: `asgi.py` is the production entrypoint; keep WSGI for compatibility where needed.

---

## Testing Policy
- Runner: use Django's built-in test runner (`uv run python manage.py test`). Do not use pytest.
- Structure: place tests under each app, e.g. `<app>/tests/` and `<app>/api/<vN>/tests/`. Management command tests live under `<app>/management/commands/tests/`.
- API tests: use DRF `APITestCase` with `APIClient` for integration-style tests; use `APIRequestFactory` when calling views directly.
- Unit tests: use `django.test.TestCase` for ORM-backed logic. Prefer `factory_boy` factories (see `<app>/factories.py`) with `faker` for data.
- Mocking: use `unittest.mock.patch` for external dependencies (HTTP clients, MongoDB). Tests must not perform real network or Mongo calls.
- Coverage: `uv run coverage run manage.py test` then `uv run coverage report -m` or `uv run coverage html`. Coverage data is stored in `.coverage/` as configured.
- Background tasks: do not start workers in tests. Call task functions directly or mock them; assert side effects explicitly.
- Policy: every bug fix adds or updates a corresponding test.

---

## Linting
- Use Ruff as the linter of record: `uv run ruff check . --fix` for safe auto‑fixes.

---

## Logging & Observability
- App logging: use `logging.getLogger(__name__)` in modules. Keep logs concise and actionable; include timestamp, level, module, function, and line number.
- Handlers (prod): console for interactive use; rotating file handlers for general, errors, and request logs managed by the process manager. Retain a small number of backups.
- Test runs: silence logging in tests to reduce noise.
- Health endpoint: expose a lightweight unauthenticated `GET /health` that responds "ok" via middleware or a trivial view. Keep it fast and side‑effect free.
- Background workers: enable worker health checks and periodic heartbeats; in development (DEBUG), execute tasks synchronously (no worker process).
- Privacy: never log secrets, credentials, full tokens, or raw payloads that may include PII. Prefer redacted summaries.

---



## Git/CI
- Commits: batch related changes with conventional commits: `feat(scope): summary`, `fix`, `chore`, `refactor`, `test`, `docs`.
- CI pipeline: `lint → tests → build/package`; include coverage reporting.

---

## Agent Workflow (Always)
1. Clarify uncertainties; propose the smallest plan that moves work forward.
2. Implement targeted changes aligned with these patterns.
3. Run lint, tests, and relevant commands locally (using uv).
4. Summarize the change and suggest the next increment.
