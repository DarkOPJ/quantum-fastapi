# AGENTS.md — opencode-practice

## Architecture Decisions

- **Framework:** FastAPI, domain-first project structure (each feature is a self-contained package with its own `router.py`, `schemas.py`, `models.py`, `service.py`, `repository.py`).
- **Authentication:** JWT only — access tokens short-lived, refresh tokens rotated on use. No OAuth2 or third-party identity providers.
- **Authorization:** RBAC only (roles: `admin`, `user`, `editor`, etc.). Check `current_user.role` at the service/dependency layer. No ABAC.
- **Dependency management:** Before scaffolding a new environment, ask the user whether to use **Poetry** or **pip + requirements.txt**. Do not assume.
- **Deployment/ops conventions** are defined separately (see skill below). Follow them for CI/CD, Docker, Kubernetes, or cloud deployment work.

## Skills (`.opencode/skills/`)

This project has three custom skills. **Load and defer to them before writing FastAPI-related code:**

- `fastapi-architecture-core-development` — project structure, layering (routers/services/repositories/models/schemas), dependency injection, coding standards, feature-development workflow.
- `fastapi-security-standards` — authentication (JWT), authorization (RBAC), security hardening (CORS, rate limiting, headers), API contract design (status codes, versioning, error response shape), testing strategy, environment/configuration management.
- `fastapi-production-engineering-operations` — CI/CD pipelines, containerization (Docker multi-stage builds), Kubernetes/cloud deployment, reverse proxy/ASGI server setup (Gunicorn/Uvicorn/Nginx), observability, performance/load testing.

Do not duplicate skill contents here — consult the skill file for its domain.

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

This is an OpenCode configuration scaffold. No application code has been written yet. The `.opencode/` directory contains:
- **agents/** — 5 subagent definitions (`api-designer`, `backend-developer`, `code-reviewer`, `frontend-developer`, `websocket-engineer`)
- **skills/** — 3 FastAPI production skills (see above)
- **package.json** — depends on `@opencode-ai/plugin`
- **.gitignore** — ignores `node_modules`, `package.json`, `package-lock.json`, `bun.lock`, `.gitignore` itself

The root `.venv/` is a Python 3.12 virtualenv (pip only, no project deps yet).

No `opencode.json` exists. No application source, tests, or build tooling is present.
