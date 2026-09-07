# Planning & Environment Setup

## 1. Is FastAPI the right choice?

FastAPI (built on Starlette + Pydantic v2) is the current default for new Python APIs that aren't full Django applications: async-first, automatic OpenAPI generation, and validation-via-type-hints. Choose it for REST/CRUD backends, microservices, and GenAI/LLM-serving APIs — its async model and native streaming support make it a strong fit for all three.

## 2. Current version baseline (verify before pinning)

- FastAPI is on the 0.1xx line; Python 3.10+ required, **3.12 or 3.13 recommended** for new projects.
- Pydantic v2 is standard — v1-style `@validator`/`class Config` are legacy; use `field_validator`/`ConfigDict`.
- Application startup/shutdown uses the `@asynccontextmanager` **lifespan** pattern; `@app.on_event("startup")` is deprecated — don't use it in new code.
- Install with `pip install "fastapi[standard]"` (or via `uv`, see below) — the `[standard]` extra bundles Uvicorn and other common deps.

**Always verify the current FastAPI/Pydantic/Python version trio at build time** (pypi.org/project/fastapi, docs) — this stack ships frequently.

## 3. Package manager: uv as default

- **uv** (Astral) is the current standard for new Python projects: near-instant dependency resolution, built-in venv management, lockfile (`uv.lock`), and drop-in replacement for pip/pip-tools/virtualenv/Poetry workflows.

```bash
# init
uv init myapi && cd myapi
uv add "fastapi[standard]" "sqlalchemy[asyncio]" asyncpg alembic pydantic-settings

# run
uv run uvicorn app.main:app --reload

# lint/test
uv run ruff check .
uv run pytest
```

- **Poetry** remains a solid, more mature alternative if the team already standardizes on it (better for complex publishing workflows) — don't migrate an existing healthy Poetry project just to chase trend.
- **pip + requirements.txt/venv** is still fine for the simplest scripts/services but offers none of uv's speed or lockfile guarantees — avoid it as the default for anything production-bound.

## 4. Project scaffolding (layered / MVC architecture)

This skill set standardizes on **layered architecture** (organized by technical role, not by feature/domain):

```
app/
  main.py                 # FastAPI() app, lifespan, router registration
  core/
    config.py              # pydantic-settings, env config
    security.py             # password hashing, JWT helpers
    logging.py              # logging setup
  api/
    v1/
      routers/             # controllers — thin, one file per resource
        users.py
        items.py
      dependencies.py       # shared Depends() (auth, pagination, db session)
  services/                # business logic — one file per resource/domain concern
    user_service.py
    item_service.py
  repositories/             # data-access layer — DB queries only, no business logic
    user_repository.py
    item_repository.py
  models/                   # SQLAlchemy ORM models
    user.py
  schemas/                  # Pydantic request/response models
    user.py
  db/
    session.py               # engine + session factory
    base.py                   # declarative base
alembic/                    # migrations
tests/
pyproject.toml
```

**The layering rule**: `routers/` (controllers) call `services/`; `services/` contain business logic and call `repositories/`; `repositories/` talk to the database only, no business logic. `schemas/` (Pydantic) are what routers accept/return; `models/` (SQLAlchemy) are what repositories persist — never leak an ORM model directly out of an endpoint's response.

## 5. Environment/config management

- Use `pydantic-settings` (`BaseSettings`) to load and validate config from environment variables/`.env` — this gives you typed, validated config instead of raw `os.environ.get()` calls scattered through the code.
- `.env` for local dev only, `.gitignore`d; real secrets injected via your deployment platform's secret store (Kubernetes Secrets, cloud secret manager) in staging/prod — never bake secrets into the Docker image.
- Separate settings per environment (dev/staging/prod) via a single `Settings` class with an `environment` field driving conditional behavior (e.g., docs exposure, debug flags), rather than maintaining parallel config files.

## Definition of done for this phase
- [ ] Python 3.12+/uv-based project initialized with a lockfile.
- [ ] Layered folder structure in place (`api/routers` → `services` → `repositories`/`models`).
- [ ] `pydantic-settings`-based config, no hardcoded secrets or raw `os.environ` calls in business code.
- [ ] Lifespan context manager used for startup/shutdown, not `on_event`.
