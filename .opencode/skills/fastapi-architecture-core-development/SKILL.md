---
name: fastapi-architecture-core-development
description: Design and structure production-grade FastAPI applications using Clean Architecture, SOLID principles, and enterprise conventions. Use at the start of a FastAPI project, or when establishing/verifying project structure, domain-first folder layout, layering (routers/services/repositories/models/schemas), dependency injection, FastAPI/Starlette/Pydantic fundamentals, coding standards, and the feature-development workflow. Does not cover authentication, authorization, security hardening, environment configuration, API contract/versioning rules, or testing strategy — see fastapi-security-standards for those.
---

## Skill Identity

- **Purpose:** Empower the agent to create and structure production-grade FastAPI applications using enterprise back-end standards.
- **Scope:** Covers architecture design, project structure, module responsibilities, coding conventions, and the build-out workflow for FastAPI projects.
- **When to use:** At the start of a FastAPI project (planning stage) and throughout development when establishing or verifying structure, design patterns, and coding standards.
- **This skill owns:** Defining the overall application architecture, folder structure, module responsibilities, dependency injection mechanics, and coding conventions for FastAPI projects.
- **This skill does NOT own (see `fastapi-security-standards` instead):** Environment/configuration management, secrets handling, authentication, authorization, security hardening (CORS/CSRF/rate limiting/headers), API contract design (status codes, versioning, error response shape), and testing strategy. If a task involves those topics, defer to that skill.

## Engineering Philosophy

- **Clean Architecture:** Structure code into independent layers (e.g., routers/controllers, services/use-cases, repositories, models) to separate concerns. Each layer has a clear responsibility, improving maintainability, flexibility, and testability. Core business logic is decoupled from frameworks and UI.
- **SOLID Principles:** Follow SOLID for modular design:
  - _Single Responsibility:_ each class/module has one reason to change.
  - _Open-Closed:_ code should be open for extension (via interfaces/abstractions) but closed for modification.
  - _Liskov Substitution:_ subclasses replace base classes without altering correctness.
  - _Interface Segregation:_ depend on specific, minimal interfaces, not bulky ones.
  - _Dependency Inversion:_ high-level modules depend on abstractions (not concretes).
- **DRY (Don't Repeat Yourself):** Avoid code duplication. Reuse components (services, utilities, dependencies) instead of copy-pasting logic.
- **KISS (Keep It Simple, Stupid):** Prefer straightforward, understandable solutions over unnecessary complexity.
- **Separation of Concerns:** Keep different aspects in separate modules. E.g., request handling (routers/controllers) vs business logic (services) vs data access (repositories) vs schemas/models. Each component should focus on one concern.
- **Dependency Inversion:** Depend on interfaces or abstract classes (e.g. service and repository interfaces) rather than concrete implementations. Use FastAPI's DI (`Depends`) to inject services/repositories by interface, making code more testable and maintainable.
- **Maintainability & Scalability:** Code should be easy to extend and refactor. Layered design and SOLID practices ensure future changes (e.g. new features, database switches) don't require massive rewrites.
- **Testability:** Architect the app so that core logic is isolated from I/O and frameworks. Mock external dependencies (e.g., DB repositories) to facilitate unit and integration testing. (Actual test strategy and coverage rules live in `fastapi-security-standards`.)

## FastAPI Fundamentals

- **Starlette & ASGI:** FastAPI is built on the Starlette ASGI framework. Starlette provides async routing, middleware, request/response objects, background tasks, WebSockets, and test client features. ASGI (Asynchronous Server Gateway Interface) is the async successor to WSGI, enabling event-driven concurrency. FastAPI combines Starlette (HTTP layer) and Pydantic (data validation) into a unified framework.
- **Pydantic (v2):** Pydantic handles data validation, parsing, and schema generation via Python type hints. FastAPI uses Pydantic models for request bodies, query parameters, and response payloads. Pydantic v2 has major performance improvements (significantly faster model creation, reduced memory usage) and supports modern Python typing. Note: in Pydantic v2, settings (`BaseSettings`) moved to the separate `pydantic-settings` package.
- **Uvicorn:** Uvicorn is a fast ASGI server for Python. It serves FastAPI applications by implementing the ASGI spec, supporting HTTP/1.1 and WebSockets with high performance. In production, use Uvicorn (often behind Gunicorn or as a standalone) to run the FastAPI app.
- **Request-Response Lifecycle:** Uvicorn receives incoming HTTP requests and passes them to the FastAPI (Starlette) app via ASGI. Starlette's router matches the path and HTTP method. FastAPI then resolves dependencies and validates inputs (query, path, header, cookies, body) using Pydantic before calling the path operation function. The function returns a result (e.g., Pydantic model, dict, or Response), which FastAPI serializes to JSON (or other format) and sends back. Any cleanup or teardown (e.g., closing DB sessions from dependencies) happens after the response.
- **Dependency Injection:** FastAPI provides a powerful DI system. Use `Depends(...)` to declare dependencies (e.g. database session, settings, authentication). FastAPI automatically resolves and injects these for each request. Dependencies can be cached per-request (multiple `Depends(get_db)` calls share one DB session). Use `@lru_cache` on a dependency (e.g. settings provider) to load it once and reuse across requests.

## Enterprise Project Architecture

Adopt a domain-first, layered folder structure that scales as the application grows — inspired by Netflix's Dispatch pattern for modular, evolvable structure:

```
my_fastapi_app/
├── alembic/                 # Alembic DB migration files (versions)
├── app/                     # Application source code
│   ├── auth/                # Domain module: authentication/authorization
│   │   ├── router.py        # API routes (or controller) for auth endpoints
│   │   ├── schemas.py       # Pydantic models (DTOs) for auth
│   │   ├── models.py        # SQLAlchemy ORM classes for auth (e.g. User)
│   │   ├── service.py       # Business logic for auth (login, token gen)
│   │   ├── repository.py    # (Optional) Data access for auth (or combine in service)
│   │   ├── dependencies.py  # Dependencies (e.g. get_current_user)
│   │   ├── exceptions.py    # Custom exceptions (e.g. AuthError)
│   │   └── utils.py         # Utility functions (e.g. token helpers)
│   ├── posts/               # Another domain module (e.g. blog posts)
│   │   ├── router.py
│   │   ├── schemas.py
│   │   ├── models.py
│   │   ├── service.py
│   │   ├── repository.py
│   │   └── exceptions.py
│   ├── core/                # Core application setup
│   │   ├── config.py        # Pydantic settings for app (env vars)
│   │   ├── db.py             # Database engine and sessionmaker
│   │   ├── security.py       # Security settings (e.g. OAuth2 configs)
│   │   └── middleware.py     # App-wide middleware setup (logging, CORS)
│   ├── models/               # (Optional) global models or SQLAlchemy mixins
│   ├── schemas/               # (Optional) global Pydantic schemas
│   ├── dependencies/          # Common dependencies (e.g. get_db)
│   ├── middleware/             # Custom middleware classes (CORS, auth, etc.)
│   ├── utils/                   # General utilities and helpers (e.g. pagination)
│   └── main.py                   # FastAPI app instantiation and inclusion of routers
├── tests/                  # Test suite (mirrors app/ structure: auth, posts, etc)
│   ├── auth/
│   ├── posts/
│   └── conftest.py         # Pytest fixtures (test DB, app, etc.)
├── .env                    # Environment variable file (not committed)
├── requirements.txt        # Dependencies
└── Dockerfile, etc.        # Deployment configs (outside Python scope)
```

**Directory purposes and rules:**

- `app/` (or `src/`): Contains all application code. Within it, each feature or domain (like `auth`, `posts`) is a Python package.
- **Domain Modules (e.g. `auth/`, `posts/`):** Each has its own `router.py` (or `controller.py`) for FastAPI endpoints, `schemas.py` for Pydantic DTOs, `models.py` for ORM classes, `service.py` for business logic, optionally `repository.py` for data access abstraction, plus `dependencies.py`, `exceptions.py`, and utility files. Keep each module self-contained (imports only necessary layers). Avoid mixing features across modules.
- **Routers / Controllers:** Files (often named `router.py`) that register FastAPI routes for that module. They should only handle HTTP details (paths, parameters, response models) and call the service layer. They belong inside the domain package (not a top-level `controllers/`, except in some conventions).
- **Services:** Business logic layer (`service.py` in each module). Contains application rules, transaction boundaries, and calls to repositories. Does not handle HTTP or request/response directly.
- **Repositories/Data Access:** Classes or functions (often in `repository.py`) that encapsulate all database operations. They use SQLAlchemy sessions to query/commit. Services call these for CRUD. This abstraction decouples business logic from raw SQL/ORM code.
- **Models:** SQLAlchemy ORM models (in `models.py`) defining database tables/columns/relations. These should not contain business logic; they just map data. Pydantic schemas will be used to convert models to external DTOs.
- **Schemas:** Pydantic models (in `schemas.py`) for request and response payloads (DTOs). Keep them separate from ORM models to clearly define what data is accepted and returned.
- **Database (`core/db.py` or `database.py`):** Setup of database engine (async or sync), sessionmaker, and connection pooling. For async usage, use `AsyncSession` with `async_sessionmaker`.
- **Core/config:** Contains application settings classes (`config.py`). The specifics of settings management, secrets, and per-environment configuration are defined in `fastapi-security-standards`; this skill only owns _where_ config code lives, not its contents.
- **Dependencies:** Functions or classes for FastAPI's dependency injection (e.g. `get_db` to provide a DB session, `get_current_user` for auth). Place common ones here for reuse.
- **Middleware:** Custom middleware (in `middleware.py` or a folder) for cross-cutting concerns (CORS, security headers, request logging). Use Starlette's middleware interface. (Contents of security-related middleware belong to `fastapi-security-standards`.)
- **Utilities:** Generic helper functions (e.g. pagination logic, email sending). No business rules here; just stateless helpers.
- **Exceptions:** Custom exception classes (e.g. `NotFoundError`, `UnauthorizedError`) that services raise for error cases. Handlers will catch these and translate to HTTP responses.
- **Tests:** Parallel to `app/`, structure test modules under `tests/`. Use pytest; common fixtures in `conftest.py`. (Test strategy/coverage rules live in `fastapi-security-standards`.)

**Dependency flow:** Routers/controllers import services and schemas (they pass requests to services). Services import repositories and models (calling DB). Repositories import models and database session. Models should not import higher layers. Dependencies (like a `get_db` function) are injected into routes/services. This flow ensures one-way dependency: routers → services → repositories → database.

## Application Design Standards

- **Router responsibilities:** Define API routes/endpoints and parameter validation. Routers should be thin: they use FastAPI's type hints to parse and validate inputs (query, path, body via Pydantic schemas) and then call the appropriate service or controller. They should _not_ contain business logic. For example, `@router.post("/users")` will parse a `UserCreate` schema and delegate to a user service function. After calling the service, the router formats the response or raises HTTP exceptions as needed.
- **Service responsibilities:** Encapsulate business/use-case logic. Services perform operations on data and enforce business rules that go beyond basic schema validation. All cross-record or transactional logic belongs here. Services call repository methods to access or mutate data, perform computations, and may raise domain-specific exceptions (which routers catch and map to HTTP errors). Services do not handle HTTP or framework concerns.
- **Repository responsibilities:** Provide an abstraction over data storage. Each repository deals with a specific entity or aggregate. They translate service calls into database operations (e.g. SQL queries via SQLAlchemy). This decouples the data layer from business logic. Repositories should offer methods like `get_by_id`, `save`, `delete`, and hide ORM details. They should handle session management only in terms of receiving a session (injected via DI) and performing queries; they do not depend on FastAPI.
- **Model responsibilities:** Define the structure of data at rest (database schema via ORM classes). Models (e.g. SQLAlchemy `Base` subclasses) declare table columns and relationships. They may include simple methods (e.g. `__repr__`), but _no business logic_. They should not be used for input/output directly (that's what schemas are for).
- **Schema responsibilities:** Define API contracts (DTOs). Pydantic schemas specify the shape of request bodies and response data. They enforce field types and constraints (e.g. `email: EmailStr`). Use separate schemas for reading vs writing if needed (e.g. `UserCreate` without `id`, and `UserOut` with `id`). Schemas should not contain complex logic; they are for validation and serialization only. FastAPI uses these schemas to automatically generate OpenAPI docs.
- **Dependency responsibilities:** Provide common resources through DI (e.g. database session, settings, current user). Dependency functions or classes should be stateless and reusable. For example, a `get_db()` dependency yields a DB session, and `get_settings()` returns the app settings (typically cached with `@lru_cache` so it's created once). Dependencies can also enforce authentication or permissions and return user objects. They should encapsulate setup/teardown (e.g. closing DB sessions after request).
- **Thin routers:** Ensure routers/controllers simply validate input and invoke services. Do not implement logic (e.g. approval rules, complex calculations) in the route handler. Instead, parse input via Pydantic, then delegate to a service function.
- **Business logic separation:** Keep core logic in services (and possibly domain classes). Business rules (such as uniqueness checks, multi-record constraints) should never be in routers or Pydantic models. For instance, a uniqueness check for usernames belongs in a service method, not in the `UserCreate` schema or route.
- **Data access abstraction:** Use repository classes/methods to isolate database details. Services should not directly use ORM queries; instead, call repository methods. This allows easier testing (repositories can be mocked) and flexibility to swap data sources.
- **DTO patterns:** Use Pydantic schemas as Data Transfer Objects between layers. For example, a service might return a domain model or dict, which the router then serializes as a response using a Pydantic output schema. Input schemas validate and convert incoming JSON to Python types before service logic. DTOs prevent coupling between internal models and external API. FastAPI's tight integration with type hints means that annotating function parameters with Pydantic models automatically applies validation and OpenAPI documentation.

## Coding Standards

- **Naming Conventions:** Follow PEP8 style. Use `snake_case` for function and variable names, `CapWords` (PascalCase) for class names, and `UPPER_CASE_WITH_UNDERSCORES` for constants. Module and package names should be short, lowercase (underscores allowed).
- **File Organization:** Organize files by purpose/layer as above. Keep modules focused (one router or service per file). Avoid excessively large files; split logically.
- **Imports:** Use absolute imports for project modules (e.g. `from project.services.user_service import ...`). Group imports: standard library first, then third-party, then local modules. Avoid wildcard imports.
- **Function Design:** Functions should be small, do one thing (SRP), and have descriptive names. All public functions should have a docstring describing purpose, inputs, and outputs. Use type hints on all parameters and return types (FastAPI relies on these). Prefer returning explicit values (rather than modifying passed objects).
- **Class Design:** Each class should have a clear responsibility. For dependency classes or service objects, inject dependencies (e.g. DB session, repository) via the constructor (dependency inversion). Use methods rather than exposing internal state. Document classes with docstrings. Use dataclasses or Pydantic models for simple data containers if needed.
- **Type Hints:** Use Python type hints everywhere. FastAPI uses them for request parsing and OpenAPI schema generation. Type hints also help with editor autocomplete and catching errors early.
- **Documentation:** Write clear docstrings for all public modules, classes, and functions. Docstrings should describe the purpose, parameters, and return values. Keep code comments up-to-date. The OpenAPI docs will include summary/description from function signatures and docstrings, so ensure these are informative.
- **Linters & Formatters:** Enforce consistent style with tools like `ruff` or `flake8`. Format code with `black` or `pre-commit` hooks. This ensures a uniform codebase that is easier to review and maintain.

## Development Workflow (Build-Out Phase)

1. **Requirement Analysis:** Gather functional and non-functional requirements. Identify what endpoints and data models are needed. (Security-specific requirements, such as auth scope and rate limits, are refined under `fastapi-security-standards`.)
2. **Architecture Planning:** Based on requirements, outline the application layers, main modules, and database schema. Use Clean Architecture principles to map out routers, services, repositories, and data models. Decide on core dependencies (e.g. SQLAlchemy). Sketch the folder structure per **Enterprise Project Architecture** above.
3. **Project Initialization:** Create the project (via `fastapi` template or custom setup). Initialize a Git repo (ask first), a virtual environment (`.venv`), and a new FastAPI app. Set up the folder skeleton from **Enterprise Project Architecture**. Ask the user whether to use Poetry or pip (with `requirements.txt`) for dependency management before proceeding. Once chosen, add core dependencies (FastAPI, Pydantic, SQLAlchemy, Uvicorn, etc.) and a test framework (pytest) using that tool.
4. **Feature Development:** Implement features iteratively. For each endpoint or use-case: create/extend Pydantic schemas (input/output), add or update ORM models, write service methods with business logic, and define routes in routers that delegate to services. Always follow the separation of layers. For database changes, add migrations (if using Alembic). Typical sequence: define any new schemas, implement service logic raising domain exceptions, implement corresponding repository queries, then wire the router. Use version control branches for each feature.

_(Testing preparation and code review preparation — steps 5 and 6 of the full development lifecycle — are defined in `fastapi-security-standards`, since they depend on the security and contract rules owned there.)_

## Agent Rules

- Structure the project into distinct layers/files as defined above. Always place code in the correct folder (e.g. do not put SQLAlchemy models in the router). Follow Clean Architecture.
- **Mandatory practices:** SOLID, DRY, PEP8 naming, type hints, and thorough docstrings. Use dependency injection for all external resources (DB, settings). Validate all inputs with Pydantic schemas.
- Use FastAPI and Pydantic features: automatic validation, error handling, and OpenAPI docs. Define request/response models for all endpoints.
- **Database Access:** Always use dependency-injected async sessions. For example, call `db: AsyncSession = Depends(get_db)` in routers/services. Never create a global or new DB engine/session inside a function. Always use SQLAlchemy ORM with parameterized queries to avoid SQL injection.
- **Transactions:** Services should control transactions (commit/rollback). Repositories should not commit; they should return results to the service, which commits at the end of the use case. Use a Unit-of-Work pattern (context) if needed.
- **Anti-Patterns (Forbidden):**
  - **No Blocking Calls:** In async routes/services, do not call blocking I/O (like file or network) without using async libraries or `run_in_threadpool`. Blocking code will freeze the event loop.
  - **No Hardcoded Config:** Do not hardcode URLs, secrets, or credentials. All config must come from `settings`.
  - **No In-Router Logic:** Never implement loops, database queries, or complex logic directly in route functions.

Follow these rules strictly to ensure the generated code is structured and production-ready. For everything related to securing, configuring, contractually defining, and testing this codebase, defer to `fastapi-security-standards`.
