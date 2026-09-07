# Testing (Essentials Only)

Kept intentionally light — the goal here is solid baseline coverage, not an exhaustive QA framework.

## Core setup

- `pytest` + `pytest-asyncio` for async test support.
- `httpx.AsyncClient` (FastAPI's own recommended test client for async apps) against the app instance directly (no real network) — this is the standard pattern:

```python
@pytest.fixture
async def client():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
        yield ac

async def test_create_user(client):
    response = await client.post("/api/v1/users/", json={"email": "a@b.com", "password": "x"})
    assert response.status_code == 201
```

## What to actually test

1. **Service-layer unit tests** — business logic with the repository mocked out (`unittest.mock`/`pytest-mock`), no real database needed. This is where most of your test value comes from and where tests run fastest.
2. **Router-level integration tests** — hit real endpoints via `AsyncClient` against a **test database** (a separate Postgres instance/schema, or SQLite for speed if your queries don't rely on Postgres-specific features), covering the happy path and the most important error cases (auth failure, validation failure, not-found) per resource.
3. **Dependency overrides for auth in tests**: override `get_current_user` via `app.dependency_overrides` to inject a fake authenticated user rather than performing real login flows in every test.

## What to skip at this depth

- Full contract/load testing (Locust, schemathesis) — add only if/when performance or contract-stability becomes a demonstrated problem, not by default.
- Exhaustive edge-case matrices for every field — cover the realistic failure modes (bad input, missing auth, not-found, conflict), not every theoretically possible combination.

## Test database

- Use a dedicated test database (Docker Compose service, or an ephemeral container spun up in CI) rather than pointing tests at a shared dev database — test isolation matters more than saving a container.
- Wrap each test in a transaction that's rolled back at the end (or truncate tables between tests) so tests don't leak state into each other.

## Definition of done for this phase
- [ ] Service-layer logic has unit tests with the repository mocked — no real DB required to run them.
- [ ] Each router has integration tests covering happy path + the main error cases (401/404/422/409 as applicable).
- [ ] Tests run against an isolated test database, not a shared dev database.
- [ ] Auth is overridden via `app.dependency_overrides` in tests rather than performing real login flows everywhere.
