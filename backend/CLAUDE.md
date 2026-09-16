# CLAUDE.md

## Project Overview

Table Hopping is a FastAPI backend for a board game rental marketplace. Python 3.11, async stack (FastAPI + SQLAlchemy 2.0 + asyncpg), PostgreSQL, Redis, Alembic, JWT auth (python-jose + bcrypt), Pydantic v2. Payments via MercadoPago, push via FCM, images on Azure Blob, announcements from Contentful.

## Commands

```bash
# Install
pip install -r requirements-dev.txt

# Run API
uvicorn app.main:app --reload

# Run scheduler worker (singleton process; HTTP replicas never start APScheduler)
SCHEDULER_ROLE=worker python -m app.scheduler_main

# Quality (also available individually)
make ci                # lint + format-check + typecheck + test + security (Lefthook pre-push)
make lint              # ruff check app tests
make format            # ruff format app tests (auto-fix)
make format-check      # ruff format --check
make typecheck         # mypy app (strict mode)
make test              # pytest with coverage, --cov-fail-under=80
make security          # scripts/security_check.sh (bandit + pip-audit + gitleaks if installed)
make load-test         # Locust; acceptance/stress/stub variants in load_tests/README.md
make pre-pr            # ./scripts/pre_pr_check.sh

# Single test
pytest tests/integration/test_auth.py::TestLogin::test_login_success

# Database
docker-compose up -d db redis
alembic -c alembic.ini upgrade head
alembic -c alembic.ini revision --autogenerate -m "describe_change"
alembic -c alembic.ini downgrade -1

# Seed data
python scripts/seed_data.py          # local dev dataset; seed_data_review.py / seed_data_prod.py
python scripts/bgg_seed_games.py     # BoardGameGeek game import

# Full stack
docker-compose up -d                 # db, redis, api, scheduler
docker-compose exec api alembic -c alembic.ini upgrade head
```

**Postgres integration tests** (default test DB is SQLite in-memory; compose has no test DB service — point `TEST_DATABASE_URL` at any Postgres):
```bash
TEST_DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5433/tablehopping \
  pytest tests/integration/
```
Some tests are skipped unless `TEST_DATABASE_URL` is Postgres (enum drift checks in `tests/integration/api/test_rentals_mercadopago.py`).

**Dependencies**: `pyproject.toml` is the source of truth; export with `scripts/export_requirements.sh`.

**Git hooks**: `make install-hooks` installs Lefthook. Pre-commit runs ruff + mypy on staged files; pre-push runs `make ci`. Bypass with `LEFTHOOK=0 git push`.

## Architecture (Modular monolith + DDD)

The app is organized by **feature module**, not by global layer. Each module owns its own `api / domain / infrastructure` slice. Dependencies point inward: `api → domain ← infrastructure`. Domain code has no FastAPI, SQLAlchemy, or Pydantic imports.

```
app/
├── main.py             # FastAPI entrypoint: CORS, rate limit, request logging, health, /metrics, lifespan
├── scheduler_main.py   # Dedicated APScheduler worker process (SCHEDULER_ROLE=worker)
├── core/               # Cross-cutting infra (config, database, security, middleware, exceptions, ...)
├── composition/        # DI wiring shared across modules
├── shared/             # api/ (auth + security deps, exception mappers, common schemas),
│                       #   domain/ (exceptions, protocols), infrastructure/ (persistence base +
│                       #   model_registry, redis_cache, security implementations)
├── modules/
│   ├── router.py       # Aggregates every module router under /api
│   ├── announcements/  # Contentful-backed announcements
│   ├── auth/           # Login, refresh, password reset (Redis-backed store)
│   ├── games/          # Game catalog, categories, catalog cache
│   ├── images/         # Upload + Azure Blob storage
│   ├── notifications/  # Mail (SMTP), push (FCM), rental/publication notification ports
│   ├── publications/   # Listings
│   ├── rentals/        # Rentals, food bundles, payments: MercadoPago checkout, webhooks,
│   │                   #   refunds, retries, lifecycle/timeout schedulers
│   ├── reviews/        # Booking reviews
│   └── users/          # Profiles, device tokens
├── infrastructure/external/  # mercadopago_client.py, mercadopago_parsers.py
└── templates/          # Email templates + fonts

alembic/versions/       # Migration scripts (alembic lives at repo root; E501 ignored here)
load_tests/             # Locust suite + stub API
docs/, deploy/          # Thesis docs, Azure deployment assets
```

**Key patterns:**
- Domain layer has zero external dependencies — pure Python protocols and dataclasses
- DI via FastAPI `Depends()` chain: endpoints → services → repositories → AsyncSession; cross-module wiring lives in `app/composition/`
- All DB I/O is async (`AsyncSession`); services and repositories are `async def`
- Authorization is enforced in services via ownership checks, not just in endpoints
- Tests override dependencies via `app.dependency_overrides`
- Domain exceptions (e.g. `InvalidCredentialsError`, `DuplicateEmailError`) are mapped to HTTP responses in `app/shared/api/exception_mappers.py` and `app/core/exceptions.py`
- SQLAlchemy models are registered through `app/shared/infrastructure/persistence/model_registry.py` — a new model must be reachable from there or Alembic/scheduler will not see it
- When adding/changing an endpoint: update the router, its schemas, the mapper, and the domain service — keep the OpenAPI contract in sync
- When changing persistence models: generate an Alembic migration in the same change

**Operational subsystems worth knowing before touching related code:**
- **Redis** backs rate limiting (`core/rate_limit.py`), catalog cache (`shared/infrastructure/redis_cache.py`), and the password reset store. It is a mandatory readiness dependency.
- **Scheduler**: APScheduler jobs (`modules/rentals/infrastructure/scheduler_jobs.py`) run only when `SCHEDULER_ROLE=worker` — overdue rental reconciliation, refund retries, pending checkout sweeps.
- **MercadoPago**: checkout dispatch, webhook processing (with signature verification), refunds with retry persistence. Largest subsystem in `modules/rentals/`.
- **Health/metrics**: `/health/live`, `/health/ready`, `/health`, `/metrics` (Prometheus, gated by `METRICS_ENABLED`). Docs endpoints only in development.

## Testing

- **Unit tests** (`tests/unit/`): Domain logic, no database
- **Integration tests** (`tests/integration/`): Full stack with HTTP client and DB
- Default test DB: SQLite in-memory. Set `TEST_DATABASE_URL` for Postgres
- All tests are async (`pytest-asyncio`, `asyncio_mode="auto"`)
- Test fixtures in `tests/conftest.py`: `client`, `test_session`, `test_engine`, `test_user`, `auth_headers`, `unverified_user`, `unverified_auth_headers`, `password_reset_store`
- Autouse fixtures disable external operational dependencies (Redis, Sentry, MercadoPago) and swap in a passthrough password decryptor
- Coverage gate: 80% (`make test`)

## Environment

Config is `pydantic-settings` in `app/core/config.py`; `.env.example` is the full reference (~57 keys).

Core: `DATABASE_URL`, `POSTGRES_*`, `POSTGRES_PORT`, `API_PORT`, `SECRET_KEY` (≥32 chars), `ENVIRONMENT`, `DEBUG`, `CORS_ORIGINS`, `ACCESS_TOKEN_EXPIRE_MINUTES`, `REFRESH_TOKEN_EXPIRE_DAYS`, `REDIS_URL`, `SCHEDULER_ROLE`.

Feature blocks: rate limiting (`RATE_LIMIT_*`), catalog cache (`CATALOG_CACHE_*`), health timeouts (`HEALTHCHECK_*`), audit (`AUDIT_*`), observability (`SENTRY_*`, `METRICS_ENABLED`), mail (`SMTP_*`), Contentful (`CONTENTFUL_*`, `ANNOUNCEMENTS_CACHE_TTL_SECONDS`), Azure Blob (`AZURE_STORAGE_*`), BGG (`BGG_BEARER_TOKEN`), push (`FCM_*`), payments (`MERCADOPAGO_*`, `PAYMENT_RETURN_WEB_URL`).

Default `POSTGRES_PORT=5433` to avoid clashing with a local Postgres on 5432 — keep `DATABASE_URL` and `POSTGRES_PORT` in sync.

## Conventions

- **Naming**: Files/functions snake_case, classes PascalCase, routes kebab-case
- **Suffixes**: `Model` (SQLAlchemy), `Schema` (Pydantic), `Interface` (abstract repos/services), `Service`/`Repository` (implementations)
- **Type hints**: Strict mypy enforced. Use `str | None` syntax, `list[T]`, `dict[K, V]`
- **Formatting**: Ruff, line-length 88, rules `E,F,I,N,W,UP`. Import sorting enabled. Per-file ignores: `E501` in `alembic/versions/*`, `E402,E501` in `scripts/*.py`
- **Async**: All DB and I/O operations must be async
- **Error messages**: Default in Spanish (bilingual app)
- **Migrations**: Always pass `-c alembic.ini` (there is a local config inside `alembic/`)
- `AGENTS.md` mirrors this file for other agents — update both together
