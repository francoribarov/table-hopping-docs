---
name: fastapi-test-developer
description: >-
  Develops and reviews tests for TableHopping FastAPI backend using pytest,
  pytest-asyncio, AsyncClient, dependency overrides, and Clean Architecture
  boundaries. Use when adding or updating backend tests.
---

# FastAPI Test Developer (Table Hopping Backend)

Write reliable, readable tests for FastAPI endpoints, domain services, and
infrastructure repositories.

---

## Testing Stack (project-specific)

- `pytest`, `pytest-asyncio` (`asyncio_mode="auto"`)
- `httpx.AsyncClient` + `ASGITransport` for API integration tests
- SQLAlchemy async test DB (`sqlite+aiosqlite:///:memory:` by default)
- Dependency overrides via `app.dependency_overrides`

Primary fixtures (from `tests/conftest.py`):
- `client`
- `test_session`
- `test_user`
- `auth_headers`

---

## Test Type Decision Tree

### 1) Unit tests (`tests/unit/`)

Use when validating pure domain logic:
- Domain entity behavior
- Domain service business rules with mocked repository interfaces
- Validation rules and exception behavior

### 2) Integration tests (`tests/integration/`)

Use when validating full FastAPI request flow:
- Endpoint -> dependency -> service -> repository -> DB
- Request/response schemas and status codes
- Auth guards and dependency overrides

### 3) Repository tests (`tests/infrastructure/` or integration suite)

Use when validating SQLAlchemy queries:
- Filtering and pagination behavior
- Transaction behavior
- Mapping ORM <-> domain entities

---

## Mandatory Test Rules

1. **AAA structure** (Arrange / Act / Assert) in every test.
2. **One behavior per test** (single reason to fail).
3. **Explicit names**: `test_<unit>_<condition>_<expected_result>`.
4. **Async correctness**: use `async def` + `await` for async code paths.
5. **Domain exceptions**: assert specific exception types and key messages.
6. **No shared mutable global state** between tests.
7. **No real external services** in tests (mail, storage, redis should be faked/overridden).

---

## Integration Test Pattern

```python
import pytest


@pytest.mark.asyncio
async def test_create_resource_returns_201(client, auth_headers):
    # Arrange
    payload = {"field": "value"}

    # Act
    response = await client.post("/v1/resources", json=payload, headers=auth_headers)

    # Assert
    assert response.status_code == 201
    body = response.json()
    assert body["field"] == "value"
```

### What to always assert for endpoints

- HTTP status code
- Critical response fields (not just snapshot-equality)
- Error payload structure for failure paths
- Auth behavior (401/403 where applicable)

---

## Service Unit Test Pattern

Use repository/service interfaces as fakes or mocks:

```python
import pytest
from app.shared.domain.exceptions import ValidationError


@pytest.mark.asyncio
async def test_create_rental_rejects_past_start_date(service):
    # Arrange
    data = build_rental_create_data_in_past()

    # Act / Assert
    with pytest.raises(ValidationError):
        await service.create_rental(data)
```

Check:
- business rule enforcement
- collaborator call expectations
- returned domain entity values

---

## Dependency Override Guidance

When testing endpoints, prefer fixture-level overrides:
- override `get_db`
- override security/decryptor dependencies when needed
- override external adapters (mail/storage/password-reset store)

Always clear overrides after test (`app.dependency_overrides.clear()`), ideally in
fixture teardown.

---

## Database Test Guidance

- Keep tests deterministic; build data explicitly in Arrange.
- Avoid order-sensitive assertions unless order is contractually defined.
- For pagination tests, assert both returned items and metadata.
- Prefer small datasets for clarity.

---

## High-Value Coverage Checklist

- Success path
- Validation failure path
- Permission/auth failure path
- Not-found path
- Conflict/state-transition failure path (when applicable)

For updates to critical flows (`auth`, `rentals`, `payments`, account operations),
tests for all of the above are required.

---

## Anti-Patterns to Reject

- Calling real SMTP/Redis/Azure in test runs
- Using `time.sleep` in async tests
- Asserting only `status_code` without response body checks
- Catching generic `Exception` in tests when specific exception is expected
- Overly broad fixtures that make tests opaque

---

## Verification Commands

```bash
make lint
make format-check
make typecheck
make test
```

For targeted iteration:

```bash
pytest tests/integration/test_auth.py::TestLogin::test_login_success
```
