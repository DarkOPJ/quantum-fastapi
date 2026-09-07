# Microservices Patterns

Relevant specifically because this skill set targets microservices architecture as one of its primary use cases, alongside general REST/CRUD and GenAI serving.

## When to actually split into multiple services

Split a service when it has a genuinely independent scaling profile, deployment cadence, or ownership boundary from the rest of the system — not simply because "microservices are the standard." A single well-structured layered FastAPI service (per `01-architecture-layered.md`) handling several related resources is often the right size; splitting too early adds operational overhead (more deployments, more network calls, more failure modes) without a corresponding benefit.

Good signals to split: a component needs independent scaling (e.g., the GenAI inference path needs many more replicas than the CRUD path), a different team owns it, or it has fundamentally different reliability/latency requirements.

## Inter-service communication

- **Synchronous (HTTP/REST)**: simplest, use `httpx.AsyncClient` for service-to-service calls. Set explicit timeouts and retries (with backoff) on every outbound call — a downstream service's slowness/failure must not cascade into your own service hanging indefinitely.
- **Asynchronous (message broker)**: for events that don't need an immediate response (e.g., "order placed" triggering multiple downstream reactions), use a message broker (Redis Streams, RabbitMQ, Kafka) rather than synchronous HTTP calls — this decouples services' uptime from each other and naturally handles fan-out to multiple consumers.
- Prefer async/event-driven communication for anything that isn't a direct request-response need — it's what actually gives you the resilience/decoupling benefit microservices are supposed to provide; a system of services calling each other synchronously in a long chain has most of microservices' operational cost with little of the benefit.

## Shared code between services

- Extract genuinely shared code (a common `schemas` package for cross-service contracts, a shared `auth` validation helper, a shared logging config) into an internal package installable via `uv`/private package index — but keep this deliberately small. Over-sharing code between services re-creates tight coupling through a different mechanism (a shared library everyone must upgrade in lockstep) instead of through direct calls.
- Never share a database between services — each service should own its data and expose it only through its API/events. A shared database is the most common way "microservices" quietly becomes a distributed monolith with all the coupling and none of the deployment independence.

## Service boundaries & contracts

- Each service publishes its own OpenAPI schema (see `12-api-design-versioning-docs.md`) as the source of truth for its contract — consuming services/teams generate clients from it rather than relying on tribal knowledge of the API shape.
- Version each service's API independently; one service's `v2` release shouldn't force every consumer to update simultaneously (see versioning guidance in `12-api-design-versioning-docs.md`).

## Resilience patterns

- **Timeouts on every outbound call** — no unbounded waits on another service.
- **Retries with backoff** for transient failures, but bounded (don't retry indefinitely) and only for idempotent operations.
- **Circuit breaking** (or at minimum, aggressive timeouts + fallback behavior) for a downstream dependency that's degraded, so one struggling service doesn't take down every service that calls it.
- **Health checks per service** (see `11-deployment-docker-k8s.md`) so orchestration can route around/restart unhealthy instances independently.

## Observability across services

- Propagate the request/correlation ID (see `08-error-handling-middleware.md`) across service boundaries in outbound call headers — without this, tracing a single user request across multiple services in logs is effectively impossible. (Full distributed tracing via OpenTelemetry is out of scope per this skill's chosen depth, but correlation-ID propagation is the low-cost baseline every microservices setup should still have.)

## Definition of done for this phase
- [ ] Service boundaries drawn around genuine scaling/ownership/reliability differences, not split by default.
- [ ] Each service owns its own database; no direct cross-service DB access.
- [ ] Outbound inter-service calls have explicit timeouts and bounded, idempotent-only retries.
- [ ] Correlation ID propagated across service boundaries in outbound requests.
- [ ] Each service's API contract is independently versioned and documented via its own OpenAPI schema.
