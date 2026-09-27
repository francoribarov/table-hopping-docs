---
name: create-endpoint
description: >-
  Scaffolds and wires a new FastAPI backend feature across endpoint, schema,
  service interface, service implementation, repository interface, repository
  implementation, dependency providers, and tests following TableHopping Clean
  Architecture conventions.
---

# Create Endpoint (Table Hopping Backend)

Create a new backend capability with minimal, coherent edits across layers:
- API endpoint + schemas
- domain service interface + implementation
- domain repository interface (if needed)
- infrastructure repository implementation (if needed)
- dependency providers
- tests

---

## Preconditions

Before coding, confirm:
1. Feature scope and route path
2. Auth requirement (`CurrentUser` required or public)
3. Required persistence changes (if any)
4. Response contract and status codes

If persistence schema changes are required, include Alembic migration steps.

---

## Required Architecture Flow

`endpoint -> service interface -> service implementation -> repository interface -> repository implementation`

Rules. Place new code in the owning module. There is no global `app/api/` or `app/domain/`.
- Router and endpoint functions in `app/modules/<feature>/api/router.py`
- Pydantic models in `app/modules/<feature>/api/schemas.py` (`*Schema` suffix)
- Service interface in `app/modules/<feature>/domain/service_interface.py`
- Service implementation in `app/modules/<feature>/domain/service.py`
- Repository interface in `app/modules/<feature>/domain/repository.py`
- Repository and SQLAlchemy models in `app/modules/<feature>/infrastructure/`
- Module providers in `app/modules/<feature>/api/dependencies.py`
- Cross-module providers in `app/composition/dependencies.py`
- Register a new model module in `app/shared/infrastructure/persistence/model_registry.py`

---

## Implementation Steps

### Step 1: API Contracts

1. Add/extend request/response schemas.
2. Add endpoint function in target router.
3. Validate request with schema, not ad-hoc dict handling.
4. Return schema via `model_validate(...)` when mapping domain objects.

### Step 2: Domain Service

1. Define/update method on service interface.
2. Implement method in domain service.
3. Keep business rules in service layer, not endpoint layer.
4. Raise domain exceptions for business errors.

### Step 3: Repository Layer (if needed)

1. Define method in repository interface.
2. Implement async method in infrastructure repository.
3. Map ORM models to domain entities via mappers.
4. Keep SQLAlchemy specifics out of domain layer.

### Step 4: Dependency Injection Wiring

1. Ensure the repository provider exists in `app/modules/<feature>/api/dependencies.py`.
2. Ensure the service provider exists there, or in `app/composition/dependencies.py` when other modules need it.
3. Add an alias type (`Annotated[..., Depends(...)]`) when the module already uses that style.

### Step 5: Router Registration

If the endpoint is a new router, include it from `app/modules/router.py`. `app/main.py` already mounts that router at `/api`.

### Step 6: Tests

1. Add/extend integration tests for endpoint behavior.
2. Add unit tests for non-trivial business rules.
3. Cover success + validation/permission/not-found paths.

---

## Definition of Done

- Endpoint reachable and documented in OpenAPI
- All added I/O is async
- Domain purity preserved (`app/modules/<feature>/domain/` without FastAPI, SQLAlchemy, or Pydantic imports)
- Dependency chain resolves correctly via `Depends`
- Tests added/updated
- `make lint`, `make format-check`, `make typecheck`, `make test` all pass

---

## Anti-Patterns to Avoid

- Direct DB access in endpoint functions
- Returning ORM models directly from endpoint layer
- Putting business logic in repository instead of service
- Bare `except:` blocks
- Missing type hints on public methods
- Sync calls in async request path
