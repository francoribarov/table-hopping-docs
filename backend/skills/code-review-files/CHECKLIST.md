# Code Review Checklist — TableHopping Backend

Quick-reference checklist. Full authority in `CLAUDE.md`.

---

## Architecture

- [ ] **Layer boundaries:** `api -> domain <- infrastructure`, cross-cutting in `core/`
- [ ] **Domain purity:** no FastAPI, SQLAlchemy, or infrastructure imports in `app/domain/`
- [ ] **API depends on interfaces/services** — avoid direct persistence logic in endpoints
- [ ] **Infrastructure stays behind interfaces** in `app/domain/repositories/`
- [ ] **No circular dependencies** between API, domain, and infrastructure

## Async and I/O

- [ ] **DB and external I/O are async** (`async def` + `await`)
- [ ] **No blocking operations** in request path (`time.sleep`, sync HTTP clients, sync DB calls)
- [ ] **AsyncSession usage** follows transaction boundaries and commit/rollback behavior
- [ ] **Background tasks** are safe and do not leak request-scoped state

## Typing and Data Contracts

- [ ] **Type hints on public functions/methods** (mypy strict compliance)
- [ ] **Modern typing syntax:** `str | None`, `list[T]`, `dict[K, V]`
- [ ] **Pydantic request/response schemas** are explicit and stable
- [ ] **No leaking ORM models in API responses** — use schemas/domain objects
- [ ] **Naming conventions:** `Model`, `Schema`, `Interface`, `Service`, `Repository`

## Error Handling and Security

- [ ] **Domain exceptions** used for business rules, mapped to HTTP in exception mappers
- [ ] **No bare `except:`** and no swallowed exceptions without logging/context
- [ ] **Auth-sensitive endpoints** enforce user context and permission checks
- [ ] **No secrets in code** (tokens, keys, credentials)
- [ ] **Error messages** remain aligned with project language conventions (Spanish default)

## Persistence and Migrations

- [ ] **Repository implementations** stay in `app/infrastructure/persistence/repositories/`
- [ ] **Mapper logic** stays in `app/infrastructure/persistence/mappers/`
- [ ] **Schema changes include Alembic migration** when required
- [ ] **Alembic commands use explicit config:** `alembic -c alembic.ini ...`

## Testing and Quality Gates

- [ ] **Tests cover changed behavior** (unit and/or integration as appropriate)
- [ ] **Dependency overrides** used correctly in API integration tests
- [ ] **Lint:** `make lint` passes
- [ ] **Format check:** `make format-check` passes
- [ ] **Typecheck:** `make typecheck` passes
- [ ] **Tests:** `make test` passes
