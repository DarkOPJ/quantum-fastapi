---
name: fastapi-development
description: Comprehensive, current (2026) guidance for building production FastAPI backends — REST/CRUD APIs, microservices, and GenAI/LLM-serving APIs — using a layered (MVC-style) architecture, async SQLAlchemy 2.0 + Alembic, uv-based tooling, and Docker/Kubernetes deployment. Use this skill whenever the user is starting a new FastAPI project, structuring or refactoring a FastAPI codebase, choosing auth/database/task-queue approaches, building streaming or LLM-serving endpoints, setting up CI/CD, or asking any narrower FastAPI question (e.g. "how should I structure this," "SSE vs WebSockets," "Celery vs ARQ," "how do I stream an LLM response"). Don't wait for a full "build me an API" request — a single focused question should also trigger this skill.
---

# FastAPI Development — Comprehensive Skill

Current as of mid-2026 (FastAPI 0.13x line, Python 3.12/3.13, Pydantic v2 — verify exact versions before pinning, since FastAPI ships frequently).

**How to use this skill**: read this file for the map and the scope this skill was built for, then open only the `references/*.md` file(s) relevant to the current task. Each reference file ends with its own "Definition of done" checklist.

## Scope this skill was built for

This skill set was deliberately scoped around specific choices rather than covering every possible FastAPI stack exhaustively:
- **Use cases**: general REST/CRUD backends, microservices, and GenAI/LLM-serving + streaming APIs.
- **Architecture**: **layered/MVC** (routers → services → repositories/models), not domain/feature-first.
- **Database**: SQLAlchemy 2.0 async + Alembic as the default.
- **Package manager**: `uv` as the default, others covered briefly.
- **Auth**: both built-in JWT/OAuth2 and third-party providers, with a decision framework.
- **Background tasks**: dedicated queues (Celery/Taskiq/ARQ), not just `BackgroundTasks`.
- **Streaming**: SSE as the default for LLM/GenAI responses, WebSockets only when genuinely needed.
- **Deployment**: Docker/Kubernetes.
- **Testing & observability**: kept intentionally light (pytest essentials; basic structured logging, no tracing/metrics stack) — extend these two areas if a project's needs grow beyond that baseline.

## Reference map

| Topic | File |
|---|---|
| Is FastAPI right, uv/project setup, layered folder structure, config/secrets | `references/00-planning-and-setup.md` |
| The layered architecture itself: routers/services/repositories, DI wiring, anti-patterns | `references/01-architecture-layered.md` |
| Pydantic v2 schemas, validation, settings | `references/02-pydantic-schemas-validation.md` |
| Routing, `Annotated`/`Depends()` DI, versioning, async vs sync handlers | `references/03-routing-dependency-injection.md` |
| SQLAlchemy 2.0 async, repository pattern, Alembic migrations | `references/04-database-sqlalchemy-alembic.md` |
| Auth (built-in JWT/OAuth2 vs third-party), CORS, rate limiting, secrets | `references/05-auth-security.md` |
| `BackgroundTasks` vs Celery/ARQ/Taskiq, decision framework | `references/06-background-tasks-queues.md` |
| **SSE/streaming for LLM responses, GenAI serving specifics** | `references/07-streaming-genai.md` |
| Exception handling, global handlers, middleware, request-ID | `references/08-error-handling-middleware.md` |
| Testing essentials (pytest, httpx AsyncClient) | `references/09-testing.md` |
| Basic structured logging | `references/10-logging.md` |
| Docker + Kubernetes deployment, Uvicorn/Gunicorn worker model | `references/11-deployment-docker-k8s.md` |
| REST conventions, API versioning, OpenAPI docs | `references/12-api-design-versioning-docs.md` |
| Service boundaries, inter-service communication, resilience patterns | `references/13-microservices-patterns.md` |
| CI/CD pipeline, migrations-in-pipeline, dependency updates | `references/14-cicd-and-maintenance.md` |

## Quick-decision cheatsheet

- **New service, any use case in scope** → uv + layered structure (`00`, `01`) + SQLAlchemy 2.0 async + Alembic (`04`).
- **Auth**: internal/service-to-service or full control needed → build JWT/OAuth2 yourself. Consumer app needing social login/MFA fast → third-party provider. See `05` for the full framework.
- **Background work**: short/best-effort → `BackgroundTasks`. Anything durable, retryable, or GenAI-workload-shaped → a dedicated queue; async-native workloads lean Taskiq (or ARQ if its maintenance status checks out at build time) over Celery unless the org already runs Celery. See `06`.
- **Streaming an LLM response** → SSE with a consistent internal JSON event schema, not WebSockets, not raw provider-format passthrough. See `07` — this is the most detail-sensitive part of this skill set, read it in full for any GenAI-serving work.
- **Deploying to Kubernetes** → one Uvicorn process per pod, scale via replicas, not Gunicorn multi-worker inside each pod (see `11` for the reasoning and the non-K8s exception).
- **Should this be split into multiple services?** → only for genuine scaling/ownership/reliability differences, not by default. See `13`.

## Operating principles across every phase

1. **Layering is non-negotiable**: routers stay thin, services hold business logic and never import `fastapi`, repositories only touch the database. Every reference file assumes this boundary.
2. **Verify version/ecosystem-status claims before relying on them** — this stack (FastAPI, Pydantic, SQLAlchemy, and especially the async task-queue landscape) moves fast enough that "current best practice" drifts within months. Where a reference file flags something as "verify current status" (e.g., ARQ's maintenance state), actually check before committing a new project to it.
3. **Streaming/GenAI endpoints get the same production rigor as everything else**: timeouts, concurrency limits, client-disconnect handling, and proxy-buffering checks are not optional extras for these routes.
4. **Testing and observability are intentionally light in this skill** — don't read that as "unimportant," read it as "this project's chosen floor." If a project's stakes grow (regulated data, high-scale production traffic), revisit `09` and `10` and add contract testing/tracing deliberately rather than assuming the light baseline still fits.

## Source of truth for freshness

For anything version- or ecosystem-status-sensitive, check current official sources before answering definitively: `fastapi.tiangolo.com`, `docs.pydantic.dev`, `docs.sqlalchemy.org`, `alembic.sqlalchemy.org`, `docs.astral.sh/uv`, and the relevant task-queue project's own repo/docs for current maintenance activity.
