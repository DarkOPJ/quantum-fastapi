# Deployment: Docker & Kubernetes

## Container image

- Multi-stage `Dockerfile` using `uv` for fast, reproducible builds:

```dockerfile
FROM python:3.12-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
RUN useradd -m appuser && chown -R appuser /app
USER appuser
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- Run as a **non-root user** in the container (a common, easily-missed hardening step).
- Keep the image slim (`python:3.12-slim`, multi-stage build so build-time dependencies don't ship in the final image).

## Process model: Uvicorn alone vs Gunicorn+Uvicorn workers, on Kubernetes specifically

This matters and is commonly gotten wrong:

- **On Kubernetes, prefer one Uvicorn process per container/pod, and scale via Kubernetes replicas** (`Deployment` with multiple pods), not by running multiple Gunicorn-managed Uvicorn workers inside a single pod. Kubernetes already handles process-level replication, restart-on-crash, and load distribution across pods — stacking Gunicorn's own multi-worker process management underneath it is usually redundant complexity fighting the orchestrator rather than complementing it.
- **Gunicorn + `UvicornWorker`** (`gunicorn app.main:app --workers N --worker-class uvicorn.workers.UvicornWorker`) is the right pattern for **non-Kubernetes single-server deployments** (a plain VM, a single Docker host) where you need Gunicorn's process supervision and multi-worker management because there's no orchestrator doing that job for you.
- If you do want multiple workers per pod for cost reasons (fewer, denser pods), that's a legitimate choice too — just make it deliberately (worker count tied to `CPU limit`, not an arbitrary number) rather than defaulting to it out of habit.

## Kubernetes manifest essentials

- **Resource requests/limits**: set both CPU and memory requests/limits explicitly — unbounded pods can starve neighbors or get OOM-killed unpredictably.
- **Readiness probe**: a `/healthz` (or similar) endpoint that checks the app can actually serve traffic (DB connection alive) — Kubernetes routes traffic only to pods passing this.
- **Liveness probe**: a simpler check confirming the process hasn't deadlocked — Kubernetes restarts the pod if this fails repeatedly. Keep liveness lighter than readiness (don't fail liveness on a slow DB — that causes unnecessary restarts; that's what readiness is for).
- **Graceful shutdown**: handle `SIGTERM` (Uvicorn does this by default) so in-flight requests complete before the pod terminates; set `terminationGracePeriodSeconds` to comfortably exceed your longest expected request (relevant for streaming/GenAI endpoints specifically — see `07-streaming-genai.md`).

```yaml
readinessProbe:
  httpGet: { path: /healthz, port: 8000 }
  initialDelaySeconds: 5
  periodSeconds: 10
livenessProbe:
  httpGet: { path: /healthz/live, port: 8000 }
  initialDelaySeconds: 10
  periodSeconds: 15
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits: { cpu: "1", memory: "512Mi" }
```

## Reverse proxy / ingress

- Terminate TLS at the ingress/load balancer, not in Uvicorn.
- For streaming (SSE) endpoints specifically, verify the ingress controller isn't buffering responses (see `07-streaming-genai.md`) — this is a common silent failure mode after moving a working local SSE endpoint into a Kubernetes ingress.

## Definition of done for this phase
- [ ] Multi-stage Dockerfile using `uv`, running as non-root, slim base image.
- [ ] Process model (plain Uvicorn + K8s replicas vs Gunicorn multi-worker) chosen deliberately based on whether Kubernetes is doing the orchestration.
- [ ] Readiness and liveness probes configured with the correct distinction (readiness checks dependencies, liveness stays lightweight).
- [ ] Resource requests/limits set explicitly; graceful shutdown verified for in-flight (especially streaming) requests.
- [ ] Ingress/proxy buffering explicitly checked for any SSE/streaming routes.
