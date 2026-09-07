# Layered (MVC-style) Architecture

## Why layered, not domain/feature-first, here

This skill set deliberately teaches **layered architecture**: code organized by *technical role* (routers/controllers, services, repositories/models) rather than by *feature/domain* (a folder per bounded context). Layered is simpler to onboard into, matches how most FastAPI tutorials and boilerplates are structured, and is the right default for small-to-mid-sized services and most microservices where a single service maps to a small, cohesive set of resources.

If a single service grows to genuinely many unrelated resource groups, consider splitting it into multiple services (see `13-microservices-patterns.md`) rather than switching the internal structure to domain-first — in a layered/microservices world, "the service boundary" does the job that "the domain folder" would otherwise do.

## The three layers

### 1. Routers (controllers) — `api/v1/routers/`
- Thin. Parse/validate request (FastAPI + Pydantic does this via type hints), call exactly one service method, return a Pydantic response schema.
- No business logic, no direct DB access, no direct SQLAlchemy imports.
- One router module per resource (`users.py`, `items.py`), registered onto the app via `APIRouter` + `include_router` — never put all routes in `main.py`.

```python
# api/v1/routers/users.py
router = APIRouter(prefix="/users", tags=["users"])

@router.post("/", response_model=UserRead, status_code=status.HTTP_201_CREATED)
async def create_user(
    payload: UserCreate,
    service: Annotated[UserService, Depends(get_user_service)],
) -> UserRead:
    return await service.create_user(payload)
```

### 2. Services — `services/`
- Business logic lives here: validation beyond what Pydantic can express, orchestration across multiple repositories, transaction boundaries, calling external APIs.
- Services depend on repository **interfaces**, not on SQLAlchemy directly — this is what keeps services testable without a real database (mock the repository in unit tests).
- Services raise domain-specific exceptions (e.g., `UserAlreadyExistsError`); routers translate those into HTTP exceptions (see `08-error-handling-middleware.md`) — services should never import `fastapi.HTTPException` directly, or you couple business logic to the web framework.

### 3. Repositories & Models — `repositories/`, `models/`
- Repositories contain **only** data-access code: queries, inserts, updates via SQLAlchemy's async session. No business rules here beyond simple query construction.
- Models (`models/`) are SQLAlchemy ORM classes — internal to the data layer. They must never be returned directly from a router; always map to a Pydantic schema (`schemas/`) first, so your API contract stays stable even if the DB schema shifts.

## Dependency injection wiring

FastAPI's `Depends()` + `Annotated` is the DI mechanism at every layer boundary — no separate service-locator/DI container needed for most apps:

```python
# api/v1/dependencies.py
async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        yield session

DbSession = Annotated[AsyncSession, Depends(get_db_session)]

def get_user_repository(db: DbSession) -> UserRepository:
    return UserRepository(db)

def get_user_service(
    repo: Annotated[UserRepository, Depends(get_user_repository)]
) -> UserService:
    return UserService(repo)
```

Chain dependencies (`get_db_session` → `get_user_repository` → `get_user_service` → router) rather than instantiating services/repositories manually inside route handlers — this is what makes every layer independently swappable in tests (override any `Depends()` with `app.dependency_overrides[...]` in `TestClient`/`AsyncClient`).

## Common anti-patterns to catch in review

- A router with a raw SQLAlchemy query in it (skipping the repository layer).
- A service importing `HTTPException` or anything from `fastapi` (couples business logic to the web layer — makes services impossible to reuse in a CLI/worker context, e.g. a Celery task).
- A global `db` variable/module-level session used instead of `Depends(get_db_session)` — breaks connection pooling correctness and testability.
- Returning a SQLAlchemy model instance directly as a response (leaks internal fields, breaks `response_model` validation guarantees).

## Definition of done for this phase
- [ ] Every router is thin: parse → call one service method → return schema.
- [ ] No `fastapi` imports anywhere under `services/` or `repositories/`.
- [ ] No raw SQLAlchemy queries outside `repositories/`.
- [ ] All DI wiring goes through `Depends()`/`Annotated` chains, no manual instantiation or globals.
