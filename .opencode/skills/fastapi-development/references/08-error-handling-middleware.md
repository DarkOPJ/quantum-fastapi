# Error Handling & Middleware

## Exception handling strategy

- Services raise **domain-specific exceptions** (`UserNotFoundError`, `InsufficientBalanceError`), never `fastapi.HTTPException` — this keeps business logic reusable outside the web layer (CLI scripts, worker tasks).
- Register global exception handlers in `main.py` that translate domain exceptions into consistent HTTP responses:

```python
@app.exception_handler(UserNotFoundError)
async def user_not_found_handler(request: Request, exc: UserNotFoundError):
    return JSONResponse(status_code=404, content={"detail": str(exc)})
```

- This gives you **one place** to control the API's error response shape (consistent `{"detail": ..., "error_code": ...}` structure) instead of reformatting errors ad hoc in every router.
- Never let an unhandled exception leak a stack trace or internal detail to the client in production — add a catch-all handler for unexpected exceptions that logs the full error server-side and returns a generic 500 to the client.

## Validation errors

- FastAPI/Pydantic already returns well-formed `422 Unprocessable Entity` responses for request validation failures — don't intercept and reformat these unless you have a specific API contract requirement to change the shape; the default is already client-friendly.

## Middleware

- **CORS** (`CORSMiddleware`): explicit origins per environment (see `05-auth-security.md`).
- **Request ID / correlation ID**: generate or propagate an `X-Request-ID` header, attach to `request.state`, include in every log line for that request — essential for tracing a single request through logs in production (see `10-logging.md`).
- **Timing/logging middleware**: log method, path, status code, and duration for every request — the minimum viable observability layer even without full tracing.
- **Order matters**: middleware executes in the order added for the request path and reverse order for the response path — keep CORS and request-ID middleware near the outside (added first) so they wrap everything else.

```python
@app.middleware("http")
async def add_request_id(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID", str(uuid4()))
    request.state.request_id = request_id
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response
```

## Definition of done for this phase
- [ ] Domain exceptions (not `HTTPException`) raised from services; translated to HTTP responses via registered global exception handlers.
- [ ] A catch-all handler prevents unhandled exceptions from leaking stack traces/internals to clients.
- [ ] Request-ID middleware in place, propagated into logs.
- [ ] CORS explicitly configured per environment, not left as a permissive default.
