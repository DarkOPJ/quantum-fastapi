# Logging (Basic — No Tracing/Metrics)

Scope deliberately limited to structured logging; tracing/metrics (OpenTelemetry, Prometheus) are out of scope for this skill by design.

## Structured logging setup

- Configure Python's standard `logging` module to emit **structured (JSON) logs** in production — plain text logs are hard to query/filter once you have any real log volume; JSON logs work cleanly with most log aggregation platforms (CloudWatch, Datadog, Loki, etc.) without extra parsing.
- Configure once in `core/logging.py`, called from the lifespan startup — not scattered `logging.basicConfig()` calls.

```python
# core/logging.py
import logging, sys
from pythonjsonlogger import jsonlogger  # or a lightweight hand-rolled JSON formatter

def configure_logging(environment: str) -> None:
    handler = logging.StreamHandler(sys.stdout)
    if environment == "prod":
        handler.setFormatter(jsonlogger.JsonFormatter())
    else:
        handler.setFormatter(logging.Formatter("%(asctime)s %(levelname)s %(name)s: %(message)s"))
    logging.basicConfig(level=logging.INFO, handlers=[handler])
```

- Log to **stdout/stderr**, not to files inside the container — let the container platform (Docker/Kubernetes) handle log collection/routing; writing to files inside an ephemeral container loses logs on restart and complicates rotation for no benefit.

## What to log

- Every request: method, path, status code, duration, and the request ID (from the middleware in `08-error-handling-middleware.md`) — this alone covers most day-to-day debugging needs.
- Every unhandled exception, with the full traceback, at `ERROR` level, tagged with the request ID.
- Key business events at `INFO` (user created, order placed) — useful for debugging without needing a full analytics pipeline.
- **Never log**: passwords, tokens, full request bodies containing PII, API keys/secrets — even at `DEBUG` level, since debug logs have a way of ending up in production during an incident.

## Log levels in practice

- `DEBUG`: verbose, local-dev only, disabled in staging/prod by default.
- `INFO`: request lifecycle, key business events — the default production level.
- `WARNING`: recoverable issues worth knowing about (a retried external call, a deprecated field still in use).
- `ERROR`: unhandled exceptions, failed operations that need attention.

## Definition of done for this phase
- [ ] Logging configured once centrally (via lifespan startup), emitting JSON in production.
- [ ] Every log line includes the request ID where applicable, for correlating a request across log entries.
- [ ] No secrets/PII/tokens ever logged, at any level.
- [ ] Logs go to stdout/stderr only — no in-container file logging relied upon.
