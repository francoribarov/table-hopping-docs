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

Rules:
- Endpoint files in `app/api/endpoints/`
- Pydantic models in `app/api/schemas/` (`*Schema` suffix)
- Service interfaces in `app/domain/service_interfaces/`
- Service implementations in `app/domain/services/`
- Repository interfaces in `app/domain/repositories/`
- Repository implementations in `app/infrastructure/persistence/repositories/`
- Dependency wiring in `app/api/service_dependencies.py` and/or
  `app/api/repository_dependencies.py`

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

1. Ensure repository provider exists in `repository_dependencies.py`.
2. Ensure service provider exists in `service_dependencies.py`.
3. Add alias type (`Annotated[..., Depends(...)]`) if consistent with existing code.

### Step 5: Router Registration

If endpoint is in a new router module, wire it in `app/api/router.py`.

### Step 6: Tests

1. Add/extend integration tests for endpoint behavior.
2. Add unit tests for non-trivial business rules.
3. Cover success + validation/permission/not-found paths.

---

## Definition of Done

- Endpoint reachable and documented in OpenAPI
- All added I/O is async
- Domain purity preserved (`app/domain/` without FastAPI/SQLAlchemy imports)
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
