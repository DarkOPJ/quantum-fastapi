# CI/CD & Maintenance

## Pipeline structure

1. **Every PR**: `uv run ruff check .` (lint+format), `uv run pytest` (unit + integration tests against a CI-spun-up test database via Docker Compose/service containers).
2. **Merge to main**: build the Docker image, run migrations against a staging database (`alembic upgrade head`), deploy to staging, run a smoke test against staging.
3. **Release to production**: promote the same built image (never rebuild between staging and prod — deploy the exact artifact you tested) after migrations are reviewed and applied.

## Database migrations in CI/CD

- Run `alembic upgrade head` as an explicit pipeline step before the new app version starts receiving traffic — never rely on the app auto-migrating on startup in production (a crash mid-migration during app boot is a much worse failure mode than a dedicated, observable migration step).
- For destructive/breaking schema changes (dropping a column an old app version still reads), use the expand-contract pattern: deploy a migration that adds the new state alongside the old (expand), deploy app code that uses the new state, then a later migration removes the old state (contract) — this keeps rolling deployments safe instead of requiring simultaneous app+schema cutover.

## Dependency updates

- Use Dependabot/Renovate (or equivalent) configured for `uv.lock`/`pyproject.toml`, with CI gating so updates are validated automatically before merge.
- Treat FastAPI/Pydantic/SQLAlchemy major-version bumps as deliberate, tested upgrades (read release notes, run the full test suite, check deprecation warnings) rather than auto-merged patch bumps.

## Image tagging & rollback

- Tag Docker images with the commit SHA (not just `latest`) so any deployed version is traceable back to exact source and reproducible/rollback-able.
- Keep the previous image available and practiced-rollback-ready (Kubernetes `Deployment` rollout history, or equivalent) so a bad release can be reverted quickly without rebuilding.

## Definition of done for this phase
- [ ] CI runs lint + tests against a real (containerized) test database on every PR, blocking merge on failure.
- [ ] Migrations run as an explicit, observable pipeline step — never implicit app-startup auto-migration in production.
- [ ] Breaking schema changes use expand-contract, not simultaneous app+schema cutover.
- [ ] Images tagged by commit SHA; rollback path tested, not just assumed to work.
- [ ] Dependency updates automated with CI gating; major-version bumps treated as deliberate reviewed upgrades.
