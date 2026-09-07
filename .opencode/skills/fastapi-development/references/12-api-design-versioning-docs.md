# API Design, Versioning & Documentation

## REST conventions

- Resource-oriented URLs (`/users/{id}`, `/users/{id}/orders`), plural nouns, HTTP verbs carrying the action (`GET`/`POST`/`PATCH`/`DELETE`) — avoid verbs in URLs (`/getUser`).
- Correct status codes: `200` (success), `201` (created, with a `Location` header where relevant), `202` (accepted — background job enqueued, see `06-background-tasks-queues.md`), `204` (success, no body), `400` (bad request), `401` (unauthenticated), `403` (unauthenticated but not authorized), `404` (not found), `409` (conflict), `422` (validation error — FastAPI's default for request validation failures), `429` (rate limited), `500` (unexpected server error).
- Use `PATCH` with partial-update schemas for partial updates, `PUT` only if you genuinely mean full-resource replacement.

## Versioning

- Path-based versioning (`/api/v1/...`) is the simplest and most explicit approach — see `03-routing-dependency-injection.md`. When a breaking change is needed, add `/api/v2/...` routers rather than mutating `v1`'s contract; keep `v1` running until clients migrate.
- Non-breaking changes (adding an optional field, adding a new endpoint) don't need a version bump — reserve versioning for actual breaking changes to existing contracts.

## OpenAPI/docs

- FastAPI generates OpenAPI/Swagger UI (`/docs`) and ReDoc (`/redoc`) automatically from your route type hints and Pydantic schemas — keep them accurate by relying on `response_model` and proper request schemas rather than loose `dict`/`Any` typing, which degrades the generated docs into something unusable.
- Add `summary`, `description`, and `tags` to routes for anything client-facing/public — the default auto-generated docs are functional but bare without these.
- **Disable or gate `/docs`/`/redoc`/`/openapi.json` in production** for internal/non-public APIs (via the `environment` setting) — exposing your full API schema publicly is a minor but real information-disclosure surface for internal services; public APIs meant to be discovered should keep docs on deliberately.
- Use `Field(description=...)`/docstrings on schemas so the generated documentation is self-explanatory to API consumers, not just to people reading the source.

## Microservices-specific API design notes

(See `13-microservices-patterns.md` for the broader inter-service architecture.)
- Keep each service's public API contract stable and independently versioned — a schema change in one service shouldn't require simultaneous deploys of every service that calls it.
- Publish/share the OpenAPI schema (FastAPI exposes it at `/openapi.json` automatically) so consuming services/teams can generate typed clients rather than hand-writing HTTP calls against your API.

## Definition of done for this phase
- [ ] Consistent REST conventions (nouns not verbs, correct status codes) across all endpoints.
- [ ] API versioned in the URL path; breaking changes go into a new version rather than mutating an existing one.
- [ ] `response_model`/typed schemas used throughout so generated OpenAPI docs are accurate and complete.
- [ ] `/docs`/`/redoc` exposure decision made deliberately per environment (public API vs internal service).
