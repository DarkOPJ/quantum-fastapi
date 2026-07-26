---
name: fastapi-security-standards
description: Secure, configure, and contractually define production-grade FastAPI applications. Use when implementing JWT-based authentication (token rotation, password hashing), role-based authorization (RBAC), security hardening (CORS, CSRF, HTTPS, security headers, rate limiting, input validation, OWASP API practices), environment/configuration management (Pydantic Settings, secrets handling, multi-environment config), API contract design (REST conventions, HTTP status codes, pagination/filtering/sorting, versioning, error response format, OpenAPI documentation), and testing/code-review preparation for FastAPI projects. Assumes the project's folder structure and layering already exist — see fastapi-architecture-core-development for that.
---

## Skill Identity

- **Purpose:** Guide the agent in securing, configuring, and contractually defining production-grade FastAPI applications, and in preparing them for testing and review.
- **Scope:** Covers configuration/secrets management, authentication, authorization, security hardening, API design/contract standards, and the testing/review stages of development.
- **When to use:** Whenever implementing or auditing auth flows, security controls, environment configuration, API request/response contracts, or preparing tests and code review for a FastAPI project.
- **This skill owns:** Configuration management, secrets handling, authentication, authorization, security practices (CORS/CSRF/rate limiting/headers/HTTPS), API design standards (status codes, pagination, versioning, error shape, OpenAPI), and testing/code-review workflow steps.
- **This skill does NOT own (see `fastapi-architecture-core-development` instead):** Folder/project structure, layering responsibilities (router/service/repository/model split), FastAPI/Starlette/Pydantic fundamentals, general coding style, and the feature-build workflow. This skill assumes that foundation is already in place.
- **Related skill:** For input/output validation, injection prevention (SQL/NoSQL/OS), SSRF, mass assignment, file upload security, and deserialization safety, see `fastapi-input-validation-vulnerability-prevention`. This skill assumes inputs are validated per that skill and focuses on auth, authorization, and security hardening only.

## Configuration Management

- **Environment Variables:** Store all secrets and config (DB URLs, API keys, secrets) in environment variables or secret managers. Don't hardcode them.
- **Pydantic Settings:** Define a settings class using Pydantic's `BaseSettings` (from `pydantic-settings`) to load and validate config from env vars. Example:
  ```python
  class Settings(BaseSettings):
      DATABASE_URL: str
      SECRET_KEY: str
      ITEMS_PER_PAGE: int = 50
  settings = Settings()
  ```
  Uppercase env vars (e.g. `DATABASE_URL`) populate the corresponding attributes automatically.
- **Secrets Handling:** Treat secrets carefully. Use strong password hashing (e.g. Argon2 via `pwdlib`, or bcrypt) for any stored credentials. For other secrets, consider using an external vault or cloud secret store in production. Never log secret values or commit them to version control.
- **Multiple Environments:** Manage config per environment (development, staging, production). Use different `.env` files or env var values per environment. Common practice: have a base `Settings` class and override it per environment (e.g. via an `ENV` variable). Ensure that debug modes or docs are disabled in production.
- **Feature Flags/Profiles:** Optionally use flags or profiles (via settings) to toggle features (like OpenAPI docs) based on environment. Hide interactive docs in production (enable only in dev) for security.

## API Design Standards

- **REST Principles:** Design endpoints around resources (nouns) and use HTTP methods as actions. `GET /items` retrieves items, `POST /items` creates a new item, `GET /items/{id}` retrieves one, `PUT /items/{id}` updates, `DELETE /items/{id}` deletes.
- **Endpoint Naming:** Use clear, noun-based, lowercase and plural endpoint paths. E.g. `/users` not `/getUsers`. Nest logically (e.g. `/users/{user_id}/orders` for orders of a user). Prefer `snake_case` for query params.
- **HTTP Methods & Status Codes:** Use standard methods and status codes. 200 OK for successful GET, 201 Created for successful resource creation, 204 No Content for successful delete, 400 Bad Request for validation errors, 401 Unauthorized / 403 Forbidden for auth errors, 404 Not Found when a resource is missing, 409 Conflict for duplicate resource, 422 Unprocessable Entity for data validation errors (FastAPI's default), 500 Internal Server Error for unexpected failures. Always return a relevant `HTTPException` from routes for error cases.
- **Pagination, Filtering, Sorting:** For endpoints returning lists, implement pagination and filtering to avoid unbounded results. Common patterns: query parameters like `limit`, `offset`, `page`, or `cursor`. Provide sorting and filtering params (e.g. `?sort=price&order=asc&category=tools`). Document these in the OpenAPI schema. Enforce sensible max limits server-side; the client should never be able to request unbounded data.
- **API Versioning:** Plan for versioning (e.g. `/v1/users`). Either include version in the path or in headers. Path versioning (`/api/v1/`) is common for public APIs to allow breaking changes in the future. Clearly document the version from the start.
- **Resource Enumeration:** Resource identifiers should be non-sequential (UUID4) per the project's model standard defined in `fastapi-architecture-core-development` — this prevents attackers from enumerating resources by guessing sequential IDs (e.g. iterating `/orders/1001`, `/orders/1002`).
- **Error Responses:** Use a consistent error response format. FastAPI by default returns JSON with `detail` for `HTTPException`. Optionally define a custom error response schema (e.g. `{"error": "string", "message": "string"}`) and use exception handlers. Ensure all endpoints use the `responses=` parameter to document potential error codes in OpenAPI. Provide meaningful error messages and codes. Handle validation errors gracefully (FastAPI does this with 422 by default).
- **OpenAPI Standards:** Leverage FastAPI's built-in OpenAPI support, which auto-generates a schema from path operation function signatures and Pydantic models. Write clear docstrings and use FastAPI parameters (`description=`, `tags=`, etc.) to document each endpoint. Use Pydantic's field examples and descriptions. Ensure the generated API docs (Swagger UI, Redoc) accurately reflect the design.

## Authentication

- Implement authentication using JWT (JSON Web Tokens). The claim should be the user id.
- Issue a short-lived access token and a longer-lived refresh token on successful login. Rotate the refresh token on every use (issue a new one, invalidate the old one).
- Hash passwords with a strong algorithm (bcrypt) before storing — never store plaintext or reversibly-encrypted passwords.
- Sign JWTs with a strong secret/key (`SECRET_KEY` from settings) and a secure algorithm (HS256). Set explicit expiration (`exp` claim) on every token.
- Never send tokens in URLs. Always require HTTPS. Never log or expose token values.
- Validate the JWT (signature, expiration, claims) via a FastAPI dependency (e.g. `get_current_user`) on every protected route.

## Authorization

- Enforce Role-Based Access Control (RBAC) only — do not implement attribute-based (ABAC) or policy-engine-based authorization.
- Define a fixed set of roles (e.g. `admin`, `user`, `editor`) and check `current_user.role` at the service or dependency layer — never in the database query and never in the router.
- Do not bake role checks into database queries; perform them explicitly in service logic or a reusable dependency (e.g. `require_role("admin")`).
- Never trust user-supplied data (e.g. a role field in a request body) for authorization decisions — always derive the role from the authenticated JWT/session.

## Security Practices

- Always validate inputs with Pydantic to prevent injection attacks. Do not use `eval` or unsafe string formatting.
- Never log sensitive data (passwords, tokens, PII).
- Always enable HTTPS/TLS in production. Set security headers (CORS policy, HSTS, Content-Security-Policy) via middleware.
- Limit CORS to known origins — never use `allow_origins=["*"]` in production.
- Implement rate limiting (via middleware or an external tool) to prevent abuse.
- If using cookies for auth, ensure CSRF protection. If using Bearer token auth, CSRF protection is not typically needed.
- Follow OWASP API Security Top 10 guidance as a baseline checklist for API-specific vulnerabilities.

## Error Handling

- Do not expose internal errors or stack traces to clients.
- In services, raise domain-specific exceptions (e.g. `NotFoundError`, `UnauthorizedError`).
- In routers or a central exception handler, catch known exceptions and return clean, consistent HTTP errors.
- Use meaningful error messages, but never leak secrets, internal state, or implementation details.

## Testing & Code Review Workflow

5. **Testing Preparation:** Write tests alongside or before implementation (TDD where practical). For services and repositories, write unit tests with mocked dependencies. For endpoints, use FastAPI's `TestClient` (or `AsyncClient`) to write integration tests — e.g. `client.post("/users/")` — verifying validation, response shape, and business rules behave as expected. Include tests for error cases (400, 401, 403, 404, 409, 422). Continuously run tests to ensure stability as features evolve.
6. **Code Review Preparation:** Before merging, verify: naming conventions, documentation, and typing are correct (per `fastapi-architecture-core-development`); routers remain thin with no direct DB access in controllers; configuration is managed properly with no secrets committed to the repo; authentication and authorization are enforced at the correct layer; all new endpoints have documented status codes and error responses; tests cover new code including failure paths; PR descriptions are clear and API documentation is updated if contracts changed.

## Agent Rules

- **Authentication:** Implement JWT-only authentication (no OAuth2, no third-party providers). Access tokens short-lived, refresh tokens rotated on use. Hash passwords with bcrypt. Never send tokens in URLs; always use HTTPS.
- **Authorization:** Enforce RBAC only (no ABAC) at the service or dependency layer, never trusting client-supplied role data.
- **Security Practices:** Validate all inputs via Pydantic. No `eval` or unsafe string formatting. No sensitive data in logs. HTTPS and security headers enforced in production. CORS restricted to known origins. Rate limiting in place. Use UUID4 as the primary key type for all ORM models.
- **Error Handling:** No internal errors or stack traces exposed to clients. Domain exceptions raised in services, translated to clean HTTP errors centrally.
- **Anti-Patterns (Forbidden):**
  - **No Wildcards:** Never use wildcard imports or `allow_origins=["*"]` in production.
  - **No Unvalidated Input:** Don't trust external input; rely on Pydantic validation and explicit permission checks.
  - **No Shortcut Security:** Avoid shortcuts like skipping token expiration checks or reusing refresh tokens. Always follow security best practices (e.g., rotate refresh tokens on each use).
  - **No Skipped Tests:** Never skip writing tests, linting, or type checks to save time; never disable security checks (e.g. turning off Bandit rules) to unblock a merge.

Follow these rules strictly to ensure the application is secure, contractually well-defined, and production-ready. For folder structure, layering, and general coding conventions, defer to `fastapi-architecture-core-development`.
