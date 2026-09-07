# Background Tasks & Task Queues

## When FastAPI's built-in `BackgroundTasks` is enough

FastAPI's `BackgroundTasks` runs fire-and-forget work in-process after the response is sent (e.g., send a confirmation email, write an audit log entry). Use it only when:
- The task is short, and losing it on a process crash/restart is acceptable (no persistence/retry).
- You don't need to check status, retry on failure, or run the task on a separate worker/process.

For anything with real durability, retry, or scaling requirements — which includes most GenAI/LLM workloads (multi-second external API calls you don't want blocking a web worker) — use a dedicated task queue instead.

## Dedicated task queue landscape (verify current maintenance status before committing)

| Queue | Model | Best for | Current notes |
|---|---|---|---|
| **Celery** | Multi-broker (Redis/RabbitMQ/SQS), sync-first, mature | Complex workflows (chains/chords/canvas), large distributed systems, teams needing the most battle-tested option | Heaviest setup/dependency footprint; async support is a workaround (gevent/eventlet), not native |
| **ARQ** | Redis-only, native asyncio | Historically the default async-native pick for FastAPI | **Maintenance status is currently mixed** — some sources report it as maintenance-only/effectively unmaintained with users pointed toward SAQ or Streaq as successors, others still actively recommend it. **Verify ARQ's current commit activity/maintainer status before adopting it for a new project** — this is exactly the kind of fast-moving ecosystem detail to check fresh. |
| **Taskiq** | Redis/RabbitMQ/NATS, native asyncio, strongly typed | The current best-supported "modern async Celery alternative" for FastAPI if ARQ's maintenance status gives pause | First-class FastAPI DI-style integration, active development as of recent checks |
| **Dramatiq** | Redis/RabbitMQ, simpler than Celery | Straightforward workloads wanting less config than Celery, don't need native async | Sync-first like Celery but lighter weight |

## Decision framework

1. **Complex, multi-step workflows; already using Celery elsewhere in the org; need the most mature tooling (Flower monitoring, Canvas workflows)** → Celery.
2. **New async-native FastAPI service, want the leanest async-first integration, and ARQ's current maintenance status checks out** → ARQ (or its suggested successor SAQ/Streaq if ARQ is confirmed inactive at build time).
3. **Want async-native with more confidence in active maintenance and stronger typing/FastAPI-style DI** → Taskiq.
4. **GenAI/LLM workloads specifically** (slow, I/O-bound external API calls — summarization, scoring, generation jobs): an async-native queue (Taskiq, or ARQ/its successor) is the better architectural fit than Celery — the workload is exactly the I/O-bound, high-concurrency case async queues are built for. Reserve Celery for this use case only if the org already has Celery infrastructure/expertise in place.

## Implementation pattern (queue-agnostic)

- Enqueue from the service layer, not the router directly, so the "this creates a background job" decision lives with business logic, not the HTTP layer.
- Return a job/task ID immediately (`202 Accepted`); expose a status/result-polling endpoint (or push updates via SSE — see `07-streaming-genai.md`) rather than making the client hold a connection open.
- Design tasks to be idempotent where possible (safe to retry) — network/worker failures will cause retries in any real deployment.
- Keep task payloads small and serializable (IDs, not large objects) — fetch the actual data inside the worker from the DB rather than serializing large objects into the queue message.

## Definition of done for this phase
- [ ] Task queue choice documented with the specific tradeoff that drove it (not defaulted to Celery without considering async-native alternatives for I/O-bound/GenAI workloads).
- [ ] Long-running/critical work goes through the dedicated queue, not `BackgroundTasks`, once durability/retry/status-checking matters.
- [ ] Enqueueing happens from the service layer; routers return a job ID immediately rather than blocking on task completion.
- [ ] Tasks are idempotent/retry-safe and pass IDs rather than large payloads.
