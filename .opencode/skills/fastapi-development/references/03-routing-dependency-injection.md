# Routing, Dependency Injection & API Versioning

## Router organization

- One `APIRouter` per resource, mounted under a versioned prefix in `main.py`:

```python
# main.py
app.include_router(users_router, prefix="/api/v1")
app.include_router(items_router, prefix="/api/v1")
```

- Version the API in the URL path (`/api/v1/...`) from day one, even if you don't expect to need `/v2` soon — retrofitting versioning onto an unversioned API is far more disruptive than starting with it. When a genuinely breaking change is needed later, add a new versioned router rather than mutating the old one's contract.

## Dependency injection with `Annotated`

`Annotated[Type, Depends(...)]` is the current idiomatic style (replacing bare `Depends()` defaults) — define reusable type aliases for common dependencies:

```python
DbSession = Annotated[AsyncSession, Depends(get_db_session)]
CurrentUser = Annotated[User, Depends(get_current_user)]

@router.get("/me", response_model=UserRead)
async def read_current_user(user: CurrentUser) -> UserRead:
    return user
```

- Chain dependencies to build up context (`get_db_session` → `get_current_user` → route) rather than repeating auth/session logic per endpoint.
- Use `Depends()` with `use_cache=True` (the default) so a dependency shared across multiple `Depends()` calls in the same request (e.g., DB session used by both an auth dependency and the route itself) is only resolved once per request.

## Async vs sync route handlers

- Use `async def` for I/O-bound work (DB calls with an async driver, HTTP calls to other services, LLM API calls) — this is where FastAPI's concurrency advantage comes from.
- Use plain `def` for genuinely CPU-bound work with no async library available — FastAPI runs sync route functions in a thread pool automatically, so you don't block the event loop, but don't make everything `async def` reflexively if the underlying work is sync/blocking (e.g., a sync-only SDK) — let FastAPI's thread-pool offload handle it instead of wrapping blocking calls in `async def` incorrectly.

## Common request-handling utilities

- **Pagination**: a shared `Depends()`-based pagination dependency (`skip`/`limit` or cursor-based) reused across list endpoints, not reimplemented per router.
- **Path/query parameter validation**: use `Annotated[int, Path(gt=0)]` / `Query(...)` constraints declaratively, same reasoning as schema validation (`02-pydantic-schemas-validation.md`).
- **Request ID / correlation ID**: inject via middleware (see `08-error-handling-middleware.md`) and make available to routes via `request.state` or a dependency, for tracing a request through logs.

## Definition of done for this phase
- [ ] API is versioned in the URL path from the first release.
- [ ] Reusable `Annotated` dependency aliases defined for DB session, current user, pagination, and settings.
- [ ] No blocking/sync I/O called directly inside an `async def` route handler.
- [ ] Routers organized one-per-resource, registered via `include_router`, not defined ad hoc in `main.py`.
