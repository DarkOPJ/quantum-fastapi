---
name: fastapi-production-engineering-operations
description: Deploy and operate FastAPI applications in production. Use for CI/CD pipelines, containerization (Docker/multi-stage builds), Kubernetes and cloud deployment (AWS/Azure/GCP), reverse proxy and ASGI server setup (Gunicorn/Uvicorn/Nginx), observability (logging, metrics, tracing, alerting), performance/load testing, and incident response/operational runbooks.
---

## Skill Identity

- **Purpose:** Guide the AI agent in building and operating FastAPI applications in production, ensuring enterprise-grade reliability, security, and maintainability.
- **Scope:** End-to-end production engineering: code quality, CI/CD pipelines, containerization, deployment architecture, cloud/Kubernetes deployment, observability, performance, and operational processes.
- **Responsibilities:** Automate building, testing, and deploying FastAPI services; enforce standards (linting, typing, security scans); manage infrastructure (Docker, Kubernetes, cloud); monitor and optimize performance; handle incidents and recovery.
- **Boundaries:** Does **not** cover application business logic or domain modeling (handled by other FastAPI skills). Focus is on deployment, operations, and production readiness only. The agent should not assume responsibility for UI, frontend, or data modeling outside FastAPI.
- **Related skills:** For code layering, project structure, and coding conventions, see `fastapi-architecture-core-development`. For authentication, authorization, and API security/contract rules, see `fastapi-security-standards`. This skill assumes both are already implemented and focuses purely on shipping and running the result.

## Code Quality Standards

- Enforce consistent formatting and linting. Use [Black](https://black.readthedocs.io) to auto-format code and [Ruff](https://docs.astral.sh) for linting and import sorting. Ensure `mypy` static typing to catch type errors pre-runtime. Integrate these in pre-commit hooks and CI pipelines.
- Perform security static analysis: use [Bandit](https://bandit.readthedocs.io) to detect common Python security issues and vulnerabilities. Configure it in CI to flag insecure patterns (hardcoded secrets, unsafe eval, etc.).
- Write clean, idiomatic code following PEP8. Enforce naming conventions, modular design, and in-code documentation. Perform rigorous code reviews: ensure pull requests include updated docs, tests, and pass all linters and type checks before merging.

## CI/CD Engineering

- **Automated Testing:** Every commit or PR triggers a pipeline (GitHub Actions, GitLab CI). Run unit/integration tests (e.g. with pytest) and code analysis steps. For example, a GitHub Actions workflow can checkout code, set up Python, install deps, run pytest, then build a Docker image on each push.
- **Build Pipelines:** Build artifacts in CI (e.g. Docker images). Use multi-stage Docker builds for efficiency (see Containerization). Tag images with commit SHA or version. Use caching (actions/cache) to speed up dependencies install.
- **Deployment Pipelines:** Automate deployments using the same CI system. For example, add workflow steps to push images to a registry and update staging/production environments. Use environment-specific branches or tags. Release strategies should include versioning (semantic versioning) and change logs. Test migrations and updates in a staging environment before production.
- **Security in CI/CD:** Store secrets (API keys, registry creds) in encrypted CI secrets vaults. Do not expose them in logs. Require status checks to pass before merging to protected branches.

## Containerization

- **Docker:** Package the app in Docker containers. Use official Python base images (e.g. `python:3.x-slim`). Write production-grade Dockerfiles:
  - **Multi-stage builds:** Separate build and runtime stages. Install build tools (e.g. gcc) only in builder stage, then copy final artifacts into a minimal runtime image to reduce size and attack surface.
  - **Optimize layers:** Order `COPY`/`RUN` commands to maximize cache usage. Include only necessary files; use `.dockerignore` to exclude dev files. Minimize image size (prefer `*-slim` images) for faster CI/CD operations.
  - **Container security:** Do not bake secrets into images. Instead inject via runtime environment variables or orchestration secrets. Run containers as non-root user when possible and keep images updated.
- **Docker Compose:** For multi-container setups (app + DB + cache + etc), define services in `docker-compose.yml` for local dev and simple deployments. Use named networks and volumes for persistence.
- **Image Registry:** Push built images to a secure registry (Docker Hub, ECR, GCR, ACR). Use image scanning tools to detect vulnerabilities.

## Deployment Architecture

- **Application Server:** FastAPI is an ASGI framework requiring an ASGI server like Uvicorn. In production, use **Gunicorn** to manage multiple Uvicorn worker processes, leveraging all CPU cores. Example: `gunicorn main:app -k uvicorn.workers.UvicornWorker --workers 3`. Use the formula `(2 × CPU cores) + 1` as a baseline for worker count.
- **Reverse Proxy:** Deploy behind **Nginx** (or similar) for SSL/TLS termination, serving static files, and load balancing. Configure Nginx to proxy to Gunicorn (via UNIX socket or localhost:8000). Set appropriate proxy headers (`Host`, `X-Forwarded-For`, `X-Forwarded-Proto`).
- **TLS/SSL:** Terminate HTTPS at the proxy. Use TLS certificates (Let’s Encrypt or managed certs). Redirect HTTP to HTTPS. Set secure headers (HSTS, X-Frame-Options, etc.).
- **Environment Handling:** Follow 12-factor config: store all environment-specific settings in environment variables, not in code. Load sensitive data (DB URLs, API keys) from env. Use a settings library (e.g. Pydantic’s BaseSettings) to centralize config. Do not commit `.env` or secrets to repo.
- **Worker Configuration:** Tune worker timeouts, keepalive, and logging. Ensure logs go to stdout/stderr (for container logs) and also to files if needed. Configure graceful restarts in process manager (systemd, Supervisord) if not using containers.

## Cloud Deployment

- **Providers:** Leverage cloud platforms (AWS, Azure, GCP) and their managed services. For example: AWS Elastic Beanstalk/ECS/EKS, Azure App Service/AKS, or Google Cloud Run/GKE. Use managed databases (RDS/Cloud SQL), caches (ElastiCache/Redis), and identity services.
- **Load Balancing & Scaling:** Use cloud load balancers (AWS ALB/NLB, Azure Load Balancer, GCP Cloud Load Balancing) to distribute traffic across instances. Configure auto-scaling groups or horizontal pod autoscalers to adjust to load (scale web servers and workers). Use health checks to remove unhealthy instances.
- **Networking:** Configure VPCs/subnets, security groups or NSGs to restrict access. Run the FastAPI service in a private subnet if possible. Use NAT or IGW as needed.
- **Managed Services:** Prefer managed queues (SQS, Pub/Sub) and storage (S3, Azure Blob) for background jobs/files. Use managed identity and secret storage (AWS Secrets Manager, Azure Key Vault) for secrets.
- **IAM/RBAC:** Assign least-privilege roles to services. Do not embed cloud credentials in code.

## Kubernetes

- **Objects:** Package the app in container images and deploy to Kubernetes for container orchestration. Use **Deployments** to manage ReplicaSets (defining `spec.template` for Pods), and **Services** to expose Pods (ClusterIP/LoadBalancer).
- **Config & Secrets:** Store configuration in ConfigMaps and secrets (k8s Secrets or sealed secrets). Mount them as env vars or volumes. Do not hardcode values in manifests.
- **Helm:** Use Helm charts (or similar templating) to define Kubernetes resources. Manage different values per environment (dev, staging, prod) with values files. Ensure idempotent and parameterized deployments.
- **Production Patterns:** Enable rolling updates (no downtime) with readiness/liveness probes. Set resource requests and limits for CPU/memory. Use anti-affinity to spread pods.
- **Ingress & TLS:** Use an Ingress controller (NGINX Ingress, Traefik) for routing and TLS termination on Kubernetes. Define proper ingress rules and cert-manager for automated certs.

## Observability

- **Logging:** Implement structured (JSON) logging with clear log levels. Include request IDs/correlation IDs in logs to trace requests end-to-end. Centralize logs (e.g. ELK stack, Loki, or cloud logging).
- **Metrics:** Expose metrics (using Prometheus client) for request rates, latencies, error rates, and custom application metrics. Aggregate with Prometheus or a cloud monitoring service.
- **Tracing:** Instrument with OpenTelemetry or similar to trace requests through services and dependencies. This helps debug latency and errors across microservices.
- **Dashboards & Alerting:** Create Grafana dashboards (or use cloud equivalents) to visualize metrics (throughput, latency, CPU/memory). Set up alerts for anomalies (high error rates, latency spikes, unhealthy hosts). Follow SRE practices: define SLOs/SLA and alert on breaches.

## Performance Engineering

- **Profiling:** Use profilers (e.g. `cProfile`, py-spy) to identify slow code paths. Address bottlenecks by optimizing algorithms or adding caching (memoization, Redis).
- **Load Testing:** Perform load tests (e.g. Locust, k6) to simulate traffic and find breaking points. Use results to scale resources and tune concurrency.
- **Caching:** Introduce caching layers (in-memory caches, Redis) for expensive operations. Employ HTTP caching headers and CDN for static content.
- **Scaling Decisions:** Based on performance data, choose scaling approach (vertical vs horizontal). For I/O-bound workloads (DB calls, HTTP calls), more workers help; for CPU-bound, match workers to CPU count.

## Production Operations

- **Monitoring:** Continuously monitor system health (uptime, errors, resource usage). Use monitoring tools (Prometheus/Grafana or cloud monitors).
- **Incident Response:** Define on-call procedures and runbooks. Alert on critical failures (CPU spikes, OOMs, 5xx rates). Log and trace errors for postmortems.
- **Backups & Recovery:** Schedule regular database backups and test restoration. Use versioned storage for critical data. Plan disaster recovery drills (region failover, cold/warm backups).
- **Maintenance & Upgrades:** Regularly apply security patches (OS, dependencies). Test dependency upgrades in staging. Use canary or blue-green deployments for minimal downtime.
- **Reliability Practices:** Implement health endpoints and self-healing (restart on failure). Document and test rollback procedures. Maintain up-to-date runbooks and infrastructure-as-code.
