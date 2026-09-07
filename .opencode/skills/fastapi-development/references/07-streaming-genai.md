# Streaming Responses & GenAI/LLM Serving

## The three streaming mechanisms FastAPI supports, and when to use which

| Mechanism | Direction | Use for |
|---|---|---|
| **`StreamingResponse`** | Server → client only | Large files, raw chunked data, CSV/binary exports — general-purpose streaming that isn't specifically the SSE event format |
| **Server-Sent Events (SSE)** | Server → client only | **This is the current standard for streaming LLM/GenAI responses.** Plain HTTP, browser-native reconnect via `EventSource` (for GET-based streams), works cleanly through proxies/auth/normal HTTP semantics |
| **WebSockets** | Bidirectional | Only when the client also needs to push data mid-stream (collaborative editing, true bidirectional chat where the client interrupts/sends while the model is still generating) — meaningfully more complex to implement and operate (connection lifecycle, scaling stateful connections across replicas) |

**Given this project's scope (GenAI streaming, avoiding WebSocket complexity): use SSE as the default, with plain `StreamingResponse` as a fallback for endpoints that don't need the SSE event framing.** This is also what the industry has converged on — OpenAI's and Anthropic's own streaming APIs both use SSE (`text/event-stream`) under the hood.

## "The newest, latest" layer: standardizing the payload format on top of SSE

The transport (SSE) isn't what's newest — SSE itself is a stable, years-old W3C-era standard. What's new and actively developing in 2026 is **standardizing the JSON event schema carried inside the SSE stream**, because today every LLM provider uses its own vendor-specific event/token format on top of SSE:

- **Vercel's AI SDK "Data Stream Protocol"** is the current de facto standard adopted broadly across the JS/TS ecosystem (and increasingly consumed from non-JS backends including FastAPI) for framing token/tool-call/finish events consistently regardless of the underlying model provider.
- An **IETF Internet-Draft ("LLM-Stream")** proposing a standard wire format for LLM inference streaming over SSE was published in 2026, explicitly aiming to solve the same fragmentation problem (every middleware/orchestration tool maintaining vendor-specific parsers) at the protocol-specification level. As of research time this is a draft, not a ratified standard — track its status before treating it as stable, but its direction (standardize the *payload schema on top of SSE*, not replace SSE as the transport) is the right one to align a new FastAPI service with.

**Practical implication for a new FastAPI GenAI service**: build the SSE endpoint so it emits a **consistent, versioned JSON event schema** (e.g., `{"type": "token", "data": "..."}`, `{"type": "tool_call", ...}`, `{"type": "done", ...}`) rather than just forwarding whatever raw format your upstream LLM provider happens to emit — this insulates your frontend/clients from provider-specific format changes and positions you to adopt the emerging standard schema with a thin translation layer rather than a rewrite.

## Implementation pattern

```python
from sse_starlette.sse import EventSourceResponse  # or starlette's own SSE support if using a version that includes it

@router.post("/chat/stream")
async def stream_chat(payload: ChatRequest, llm: LlmClientDep):
    async def event_generator():
        async for chunk in llm.stream_completion(payload.messages):
            if await request.is_disconnected():
                break  # stop pulling from the LLM if the client left — avoids paying for unseen tokens
            yield {"event": "token", "data": json.dumps({"type": "token", "data": chunk})}
        yield {"event": "done", "data": json.dumps({"type": "done"})}
    return EventSourceResponse(event_generator())
```

Key details that are easy to miss:
- **Detect client disconnect** (`request.is_disconnected()`) and stop pulling from the upstream LLM immediately — otherwise you keep paying for/generating tokens nobody will see.
- **Disable reverse-proxy buffering** (`X-Accel-Buffering: no` header, and check your ingress/load balancer's own buffering config) — without this, tokens can arrive at the client all at once instead of progressively, defeating the purpose of streaming.
- The browser's native `EventSource` only supports GET; most chat endpoints need POST (to send a message body), so browser clients typically use `fetch` + a `ReadableStream` reader rather than `EventSource` directly — factor this into frontend integration, not just the FastAPI side.
- Bind the LLM client instance via the **lifespan** context manager (load/connect once at startup, store on `app.state`), not per-request — this avoids reconnecting/reinitializing an expensive client on every request.

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.llm_client = build_llm_client(settings)
    yield
    await app.state.llm_client.aclose()

app = FastAPI(lifespan=lifespan)
```

## GenAI/LLM serving specifics beyond streaming

- **Timeouts**: LLM calls can legitimately take many seconds — set explicit, generous timeouts on the upstream client, and make sure your reverse proxy/load balancer's own timeout is configured longer than the LLM call can take, or you'll see mysterious 502/504s that have nothing to do with your app code.
- **Concurrency limits**: cap concurrent in-flight LLM calls (a semaphore, or a queue — see `06-background-tasks-queues.md`) if the upstream provider has rate limits, rather than letting FastAPI's async model happily fire unlimited concurrent requests at a rate-limited API.
- **Non-streaming fallback**: some clients/integrations (server-to-server, batch jobs) don't want a stream — offer a plain non-streaming response mode from the same underlying service method, not a separate reimplementation.
- **Cost/token accounting**: log token usage per request (most providers return usage stats in the final stream event or response) for cost tracking — this is often skipped early and painful to reconstruct retroactively.

## Definition of done for this phase
- [ ] LLM/GenAI responses stream via SSE with a consistent, versioned internal JSON event schema — not raw provider-format passthrough.
- [ ] Client-disconnect detection stops upstream generation; reverse-proxy buffering explicitly disabled for streaming routes.
- [ ] LLM client initialized once via lifespan, not per-request.
- [ ] Timeouts and concurrency limits set explicitly for upstream LLM calls; proxy/LB timeout configured to exceed the LLM call's realistic max duration.
- [ ] WebSockets used only if a specific feature genuinely requires bidirectional mid-stream client input — not adopted by default.
