# AGENTS.md — Django Projects (Postgres + Mongo + OpenAI Views)

**Purpose:** This file tells the coding agent how to work inside *any* of my Django projects. Follow it verbatim. Be concise and opinionated, ask clarifying questions first, and default to safe changes.

## Startup Checklist (Agent)
- Read this file fully.
- Detect package manager and env: `pip-compile`/`pip`, `uv`, or `poetry`. Prefer **uv** if `pyproject.toml` exists, else `pip` with `requirements.txt`.
- Ask: **“What should we work on today?”** Then propose a short plan (bulleted), get confirmation, and execute.

## Tech Context (Typical Stack)
- **Django** (or Django + DRF) on **PostgreSQL** (primary) + **MongoDB** (auxiliary/log & doc store).
- **OpenAI REST API** views for data fetching/parsing; heavy/slow work goes to **Celery**.
- Infra assumptions: `.env`-based settings, **pre-commit** with `ruff`, `black`, `isort`, `mypy`, **pytest**.

---

## Commands (Agent should use when relevant)
- **Setup:** `uv sync` (preferred) or `pip install -r requirements.txt`
- **Fmt/Lint:** `ruff check . --fix`, `black .`, `isort .`, `mypy .`
- **Tests:** `pytest -q`
- **Run:** `python manage.py runserver`
- **Migrate:** `python manage.py makemigrations && python manage.py migrate`
- **Celery:** `celery -A config worker -l INFO`
- **OpenAPI docs (if DRF Spectacular):** `/api/schema/` & `/api/docs/`

> If a command is missing, create/update the appropriate config (e.g., add `pyproject.toml` [ruff/black/isort], `mypy.ini`, `pytest.ini`, `pre-commit-config.yaml`).

---

## Project Structure (Baseline)
```
project_root/
  config/                 # settings/, urls.py, asgi/wsgi.py
  apps/
    <domain_app>/         # models.py, admin.py, views.py, tasks.py, serializers.py, urls.py
  services/               # domain services, OpenAI clients, adapters
  db/
    mongo_client.py       # pymongo client factory, typed helpers
  tests/                  # pytest tests
  scripts/                # one-off utilities
  .env, .env.example
```
- **Separation:** Keep business logic in `services/`; keep views thin; tests live near code or under `tests/` by feature.
- **Settings split:** `config/settings/{base.py,dev.py,prod.py}`. Load with `DJANGO_SETTINGS_MODULE=config.settings.dev` etc.
- **ENV management:** Use `django-environ` or `pydantic-settings`. Provide `.env.example` with all required vars.

---

## Databases
- **Postgres**: The **source of truth** for transactional data. Use `psycopg[binary,pool]` / Django ORM.
- **Mongo**: For *append-only logs, cached large docs, model outputs*. Use `pymongo` (no djongo). Do **not** mix relational invariants into Mongo.
- **Transactions:** Never span Postgres + Mongo. Treat cross-DB operations as *sagas*; persist steps idempotently with retries.
- **Migrations:** Every Django model change → migration. For Mongo, evolve collections via idempotent scripts in `scripts/` and document in Git.

---

## OpenAI REST API Views
- Use `httpx` (async preferred) or `requests` for REST calls. **Never** embed API keys in code; read from settings.
- Wrap OpenAI calls behind `services/openai_client.py` with:
  - **Retry/backoff** (e.g., 429/5xx with jitter).
  - **Timeouts** (read+connect ≤ 30s).
  - **Telemetry** (duration, model, tokens if returned).
  - **Guardrails**: validate response JSON with **Pydantic** schemas before use.
- **Views:** Prefer DRF `APIView`/`GenericViewSet` with explicit request/response serializers.
- **Async:** Use async views cautiously (DB/ORM is sync). Offload heavy LLM work to **Celery**; return 202 + task id if long-running.
- **Security:** Enable CORS only for allowed origins; enforce auth/permissions; rate-limit endpoints that hit OpenAI.
- **Privacy:** Never log full prompts/responses that may contain PII; store redacted summaries in Mongo if needed.

**Example skeleton** (sync, DRF):
```python
# apps/ai/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .serializers import PromptIn, AiOut
from services.openai_client import complete_safely

class SummarizeView(APIView):
    def post(self, request):
        data = PromptIn(data=request.data); data.is_valid(raise_exception=True)
        result = complete_safely(prompt=data.validated_data["prompt"])
        out = AiOut.model_validate(result)  # Pydantic for strictness
        return Response(out.model_dump(), status=status.HTTP_200_OK)
```
> Add robust error mapping to HTTP (400 for invalid input, 502 for upstream issues, 429 for rate limits).

---

## Services & Boundaries
- **No ORMs in views.** Views call **services**; services coordinate repositories/clients.
- **Idempotency:** For endpoints that may be retried (network), support `Idempotency-Key` header.
- **Typing:** Use `from __future__ import annotations`; type *everything*. Run `mypy` in CI.
- **Configuration:** All dynamic values via settings/env; pass into services at init (DI).

---

## Testing Policy
- **pytest** + `pytest-django`; factories via `factory_boy`; `freezegun` for time.
- **Unit tests** for services/serializers; **API tests** for views; **integration** for Celery flows.
- **Golden tests** for model outputs when reasonable (store sample input→output fixtures, not raw PII).
- **Mongo tests** use a temporary DB/collection suffix; clean up per test session.
- Every bug fix must add/adjust a test.

---

## Linting, Formatting, Security
- **ruff** (rules: E/F/W, pyflakes/pycodestyle, and `ruff lint --select I,S,B` if configured), **black**, **isort**, **mypy**.
- **Bandit** for basic security checks; ensure `SECURE_*` Django settings in prod.
- **Pre-commit**: run hooks locally; CI enforces.

---

## API Design
- Version API under `/api/v1/`.
- Use explicit serializers; never return raw model instances.
- Paginate list views; filter/ordering params validated and documented.
- OpenAPI via **drf-spectacular**; keep schema valid in CI.

---

## Logging & Observability
- **structlog** or standard logging with JSON formatter. Include request id, user id, route, latency, status, error.
- Add health checks, DB pings, and Celery beat for periodic jobs.
- Sentry (or equivalent) configured for prod.

---

## Git/CI
- Conventional commits: `feat(scope): summary`, `fix`, `chore`, `refactor`, `test`, `docs`.
- Small, focused PRs. Include test results. Avoid force pushes unless requested.
- CI: run `ruff/black/isort/mypy/pytest`. Migrate DB in staging environments.

---

## Agent Workflow (Always)
1. **Clarify** uncertainties; present 2–3 options with trade-offs where relevant.
2. **Plan** briefly and get confirmation.
3. **Implement** minimal change first; prefer services over bloated views.
4. **Test** locally; ensure `pytest` passes.
5. **Document**: update README or inline docs when introducing new patterns; keep `.env.example` current.
6. **Report**: return a concise summary + follow-up suggestion.
