# AGENTS.md — opencode-practice

## Architecture Decisions

- **Framework:** FastAPI, domain-first project structure (each feature is a self-contained package with its own `router.py`, `schemas.py`, `models.py`, `service.py`, `repository.py`).
- **Authentication:** JWT only — access tokens short-lived, refresh tokens rotated on use. No OAuth2 or third-party identity providers.
- **Authorization:** RBAC only (roles: `admin`, `user`, `editor`, etc.). Check `current_user.role` at the service/dependency layer. No ABAC.
- **Dependency management:** Before scaffolding a new environment, ask the user whether to use **Poetry** or **pip + requirements.txt**. Do not assume.
- **Deployment/ops conventions** are defined separately (see skill below). Follow them for CI/CD, Docker, Kubernetes, or cloud deployment work.

## Skills (`.opencode/skills/`)

This project has four custom skills. **Load and defer to them before writing FastAPI-related code:**

- `fastapi-architecture-core-development` — project structure, layering (routers/services/repositories/models/schemas), dependency injection, coding standards, feature-development workflow.
- `fastapi-security-standards` — authentication (JWT), authorization (RBAC), security hardening (CORS, rate limiting, headers), API contract design (status codes, versioning, error response shape), testing strategy, environment/configuration management.
- `fastapi-input-validation-vulnerability-prevention` — Pydantic v2 strict validation, injection/SSRF/mass-assignment prevention, file upload security, output filtering.
- `fastapi-production-engineering-operations` — CI/CD pipelines, containerization (Docker multi-stage builds), Kubernetes/cloud deployment, reverse proxy/ASGI server setup (Gunicorn/Uvicorn/Nginx), observability, performance/load testing.
- `documentation-standards` — README/CHANGELOG/ADR structure, API/component/code-level documentation, Markdown formatting conventions. Applies across frontend, backend, and mobile work.

Do not duplicate skill contents here — consult the skill file for its domain.

## Documentation

Load and follow `documentation-standards` in two situations:

1. **Reactively** — after any change to a feature, API/interface, config/env variable, dependency, or architectural decision, or after a non-obvious bug fix. Update or create the relevant docs (README, CHANGELOG, ADR, docstrings) in the same turn as the code change.
2. **On request** — whenever the user asks to document, update, or review documentation, regardless of whether code changed this session.

Documentation output locations relative to the domain-first structure:

- `README.md`, `CHANGELOG.md`, and `docs/` (including `docs/adr/`) live at the **project root** — not inside `app/`, and not duplicated per domain module.
- Do not create a separate README or ADR folder inside individual domain packages (e.g. `app/auth/`, `app/posts/`); domain-specific notes worth keeping belong in `docs/architecture.md` or a docstring in that module's own files, not a new root-level doc per feature.
- If a domain module needs endpoint-specific documentation, it belongs in the API reference (OpenAPI schema + `docs/api/overview.md`), not a standalone file inside that module's folder.

Do not generate documentation for trivial or purely internal changes with no observable effect. Never duplicate content across docs — link to a single source of truth instead. Ask the user when anything is unclear.

## How to Investigate

Read the highest-value sources first:

- `README*`, root manifests, workspace config, lockfiles
- build, test, lint, formatter, typecheck, and codegen config
- CI workflows and pre-commit / task runner config
- existing instruction files (`AGENTS.md`, `CLAUDE.md`, `.cursor/rules/`, `.cursorrules`, `.github/copilot-instructions.md`)
- repo-local OpenCode config (`opencode.json`)

If architecture is still unclear, inspect representative code files to find entrypoints, package boundaries, and execution flow.

Prefer executable sources of truth over prose. If docs conflict with config or scripts, trust the executable source.

## What to Extract

Look for the highest-signal facts for an agent working in this repo:

- exact developer commands (run single test, single package, focused verification)
- required command order (e.g. `lint -> typecheck -> test`)
- monorepo boundaries, ownership of major directories, app/library entrypoints
- framework quirks (generated code, migrations, codegen, special env loading)
- repo-specific style or workflow conventions that differ from defaults
- testing quirks (fixtures, integration prerequisites, required services, flaky suites)
- important constraints from existing instruction files

## Repo State

This is an OpenCode configuration scaffold. If no application code has been written yet, create the project architecture from scratch following the `fastapi-architecture-core-development` skill. If application code already exists, continue building on it consistently with that skill's conventions rather than restructuring it. The `.opencode/` directory contains:

- **agents/** — 6 subagent definitions (`api-designer`, `fastapi-documenter`, `backend-developer`, `code-reviewer`, `frontend-developer`, `websocket-engineer`)
- **skills/** — 5 skills: 4 FastAPI production skills + `documentation-standards` (see above)
- **package.json** — depends on `@opencode-ai/plugin`
- **.gitignore** — ignores `node_modules`, `package.json`, `package-lock.json`, `bun.lock`, `.gitignore` itself

The root `.venv/` is a Python 3.12 virtualenv (pip only, no project deps yet).

No `opencode.json` exists. No application source, tests, or build tooling is present.
