# FastAPI — Summary Notes

Reference built from the _"Backend Development with FastAPI for AI, ML & MLOps Engineers"_ workshop deck (MLOps MENA Community). Covers the full path from "trained model in a notebook" to a production-style AI service.

---

# FastAPI API-Oriented Features

FastAPI has API-oriented features built into the framework that Flask normally requires extensions or manual implementation for.

## API Infrastructure Features

- **Request validation** → Pydantic
- **API schema** → OpenAPI
- **Interactive API documentation** → Swagger UI / ReDoc
- **Dependency injection** → `Depends`
- **Async endpoints** → `async` / `await`
- **WebSockets** → Built-in support
- **Background tasks** → `BackgroundTasks`
- **File uploads** → `UploadFile`
- **Middleware** → Built-in middleware system

## 1. Why FastAPI

A trained model sitting in a notebook can't be called by a mobile app, website, or another microservice — someone has to expose it safely, reliably, and at scale. FastAPI is built for exactly this:

- **Built on Starlette + ASGI** — async-first foundation, handles many concurrent I/O-bound requests efficiently.
- **Type hints as the API** — your function signature _is_ the contract; FastAPI reads it to validate and document automatically.
- **Pydantic for data** — request/response schemas are plain Python classes with real, automatic validation.
- **Automatic OpenAPI docs** — an interactive Swagger UI is generated from your type hints with no extra work.

**vs. Flask / Django REST Framework:** FastAPI has native async support, built-in Pydantic validation, and automatic docs — Flask needs extensions for all three, and DRF needs more boilerplate (serializers). It's not "the best framework" universally — it's the best fit for ML/LLM inference services specifically.

---

## 2. The Mental Model: API as a Remote Function Call

An HTTP endpoint is a function call that happens to cross a network boundary:

```
result = predict(features)          # local function call
POST /predict {"features": [...]}   # same idea, over HTTP
```

This is why request bodies mirror function arguments, and response models mirror return values.

---

## 3. Anatomy of an HTTP Request

Every request has six parts: **Method** (GET/POST/etc.), **URL**, **Path**, **Query Params**, **Headers**, **Body**.

### Path Parameters — identify a specific resource

```python
@app.get("/models/{model_id}")
def get_model(model_id: int):
    return {"model_id": model_id}
```

- A value embedded directly in the URL path (e.g. `/models/123`). Use when the value **identifies** a resource. FastAPI returns `422` automatically if the type doesn't match (e.g. text where an `int` is expected).

### Query Parameters — filter/modify retrieval

```python
@app.get("/predictions")
def get_predictions(limit: int = 10, offset: int = 0):
    ...
# GET /predictions?limit=10&offset=20
```

- Optional `key=value` pairs after the `?`. Use for things that modify **how** you retrieve data (pagination, filtering) — not what resource you're identifying. Production tip: cap values with `Query(le=100)` so a client can't request a huge page.

### Request Body — complex input data

```python
from pydantic import BaseModel

class PredictionRequest(BaseModel):
    age: int
    income: float
    features: list[float]

@app.post("/predict")
def predict(request: PredictionRequest):
    ...
```

- The JSON payload, parsed into a Pydantic model. This is where ML inference inputs (features, prompts, config) live — the most important part of the request for AI APIs.

### Headers — metadata, not payload

```python
from fastapi import Header
from typing import Annotated

@app.get("/models")
def get_models(x_request_id: Annotated[str | None, Header()] = None):
    ...
```

- Key-value metadata alongside the request — auth tokens, tracing/correlation IDs. Never put business data here.

**Rule of thumb:** Path = _which_ resource · Query = _how_ to retrieve it · Body = _what_ to send it · Header = metadata about the call.

---

## 4. HTTP Methods & Status Codes

| Method   | Body? | Idempotent? | Typical use                      |
| -------- | ----- | ----------- | -------------------------------- |
| `GET`    | No    | Yes         | Read a model, list predictions   |
| `POST`   | Yes   | No          | Run inference, create a resource |
| `PUT`    | Yes   | Yes         | Replace a resource entirely      |
| `PATCH`  | Yes   | No          | Partially update a resource      |
| `DELETE` | No    | Yes         | Remove a resource                |

**Why is inference `POST /predict`, not `GET /predict`?** GET requests are meant to be cacheable, idempotent, and carry no body — but inference takes a complex payload and produces a non-cacheable, side-effecting result. POST fits.

### Status codes worth knowing for AI services

| Code        | Meaning              | Typical use                           |
| ----------- | -------------------- | ------------------------------------- |
| `200`       | OK                   | Successful GET / inference            |
| `201`       | Created              | New resource (model registered)       |
| `202`       | Accepted             | Async job queued (batch inference)    |
| `204`       | No Content           | Successful DELETE                     |
| `400`       | Bad Request          | Malformed request                     |
| `401`       | Unauthorized         | Missing/invalid credentials           |
| `403`       | Forbidden            | Authenticated but not allowed         |
| `404`       | Not Found            | Model or resource doesn't exist       |
| `409`       | Conflict             | Duplicate resource                    |
| `422`       | Unprocessable Entity | **Pydantic validation failed**        |
| `429`       | Too Many Requests    | Rate limit exceeded                   |
| `500`/`503` | Server Error         | Inference crashed / model unavailable |

---

## 5. Pydantic & Validation

```python
from pydantic import BaseModel, Field

class PredictionRequest(BaseModel):
    age: int = Field(gt=0, lt=120)
    income: float = Field(ge=0)
    features: list[float]
```

- **Field constraints** (`gt`, `lt`, `ge`, `le`) enforce valid numeric ranges automatically — bad input (negative age, wrong shape) fails loudly before it ever touches the model.

```python
class User(BaseModel):
    name: str
    email: str

class PredictionRequest(BaseModel):
    user: User
    features: list[float]
```

- **Nested models** — compose complex payloads by embedding one model inside another.

**Validation pipeline:** `Raw Request → Validation → Business Logic → Model Inference`. An invalid request never reaches your model, GPU, or token budget. On failure, FastAPI automatically returns `422` with a detailed JSON body naming exactly which field failed — no custom code needed.

**Common mistake:** validating manually with `if` statements instead of letting Pydantic do it.

---

## 6. Response Models & Serialization

```python
class PredictionResponse(BaseModel):
    prediction: int
    probability: float

@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest):
    return {"prediction": 1, "probability": 0.93}
```

- `response_model` defines the **public data contract** for what clients see — preventing leaks of internal DB fields, secrets, or raw model internals.
- **Two-way validation** — Pydantic validates both input and output, so a bug can't accidentally leak an unexpected field.
- **Common mistake:** returning a raw SQLAlchemy object or plain dict instead of a defined schema.

---

## 7. Project Structure

```
app/
├── main.py
├── api/
│   └── routes/
│       ├── predictions.py
│       ├── users.py
│       └── health.py
├── schemas/       # Pydantic request/response models — your data contracts
├── models/
├── services/       # Business logic: orchestration, calling the model, calling the DB
├── repositories/    # Data access — the only layer that talks to SQLAlchemy directly
├── core/
├── dependencies/
└── db/
```

- **api/routes** — thin HTTP layer: parses requests, calls services, returns responses.
- **Why not everything in `main.py`?** Past a handful of endpoints it becomes untestable, unreadable, and impossible for a team to work on in parallel.

---

## 8. SQLAlchemy ORM

```python
class Prediction(Base):
    __tablename__ = "predictions"
    id = Column(Integer, primary_key=True)
    input_features = Column(JSON)
    result = Column(Float, nullable=False)
```

- **ORM** — maps Python classes to database tables, so you avoid hand-writing raw SQL strings everywhere. Used to persist predictions, users, and feedback for retraining/audits.
- **Migrations:** use **Alembic** to version-control schema changes — never edit a production table by hand.

```python
# Create
todo = Todo(title="Label dataset")
db.add(todo); db.commit(); db.refresh(todo)

# Read
todo = db.query(Todo).get(todo_id)

# Update
todo.completed = True
db.commit()

# Delete
db.delete(todo)
db.commit()
```

- **CRUD with a session** — inject a fresh DB session per request via dependency injection (§9), not a global. Use connection pooling, and always commit/rollback in a try/finally.

---

## 9. Dependency Injection — `Depends()`

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/todos")
def get_todos(db=Depends(get_db)):
    ...
```

```python
_model = load_model()   # loaded once, at startup

def get_model():
    return _model

@app.post("/predict")
def predict(req: PredictionRequest, model=Depends(get_model)):
    return model.predict(req.features)
```

- `Depends()` lets FastAPI inject DB sessions, loaded models, config, and auth — without global state.
- **Testing:** swap a real dependency for a mock/fake using `dependency_overrides`.
- **Common mistake:** loading a large model _inside_ the endpoint body — it reloads on every single request instead of once at startup.

_(See §13 for a deeper explanation of `APIRouter`, which pairs with dependency injection when organizing routes across files.)_

---

## 10. Concurrency & Async

```python
@app.get("/data")
async def get_data():
    result = await some_io_operation()
    return result
```

- **`async def` shines for I/O-bound work** — DB calls, HTTP requests, calling LLM providers, object storage. Analogy: a chef who starts water boiling then chops vegetables while it heats, instead of staring at the pot.
- **Async ≠ faster ML inference.** `async` does **not** automatically make CPU-heavy inference faster. A blocking `model.predict()` inside `async def` freezes every other request on the server.
- **CPU-bound work** → use a thread/process pool (`run_in_executor`, `BackgroundTasks`) or a separate worker process.

```python
# BAD — blocks the entire event loop, every other request waits
@app.post("/predict")
async def predict(request: PredictionRequest):
    result = model.predict(request.features)   # CPU-heavy, synchronous
    return result

# BETTER — offload blocking work to a thread pool
from fastapi.concurrency import run_in_threadpool

@app.post("/predict")
async def predict(request: PredictionRequest):
    result = await run_in_threadpool(model.predict, request.features)
    return result
```

- Production tip: for genuinely heavy inference, prefer a dedicated worker process/queue over stuffing everything into one API process.

---

## 11. Rate Limiting

```python
from fastapi import HTTPException

@app.middleware("http")
async def rate_limit(request, call_next):
    key = request.client.host
    count = await redis.incr(key)
    if count > 100:
        raise HTTPException(429, "Rate limited")
    await redis.expire(key, 60)
    return await call_next(request)
```

- Inference is expensive — protect it. Common strategies:
  - **Fixed window** — N requests per fixed time slot (e.g. per minute).
  - **Sliding window** — smooths bursts at window boundaries.
  - **Token bucket** — allows short bursts while enforcing an average rate.
  - **Leaky bucket** — processes requests at a constant, steady rate.

---

## 12. Middleware

```python
import time

@app.middleware("http")
async def log_requests(request, call_next):
    start = time.time()
    response = await call_next(request)
    dur = time.time() - start
    response.headers["X-Process-Time"] = str(dur)
    return response
```

- Flow: `Request → Middleware → Route → Middleware → Response`.
- **Good for:** logging, request timing, CORS, security headers, request IDs — anything applying to **all** routes.
- **Prefer `Depends()` instead when:** the logic is route-specific (auth for one group of endpoints, loading one resource).
- **Common mistake:** doing heavy per-route logic in middleware — it runs on _every single request_, so keep it cheap.

---

## 13. `APIRouter` — Organizing Routes Across Files

`APIRouter` is the piece that makes the project structure in §7 possible — it lets you define endpoints in separate files (`predictions.py`, `users.py`, `health.py`) instead of crowding every route into `main.py`.

```python
# api/routes/predictions.py
from fastapi import APIRouter

router = APIRouter()

@router.post("/predict")
async def predict(request: PredictionRequest):
    ...

@router.get("/predict/{prediction_id}")
async def get_prediction(prediction_id: int):
    ...
```

```python
# main.py
from fastapi import FastAPI
from api.routes import predictions, users, health

app = FastAPI()
app.include_router(predictions.router, prefix="/predictions", tags=["predictions"])
app.include_router(users.router, prefix="/users", tags=["users"])
app.include_router(health.router, tags=["health"])
```

**What it does and why it matters:**

- Think of `APIRouter` as a **mini FastAPI app** — it supports `@router.get()`, `@router.post()`, dependencies, and its own set of routes, but isn't runnable on its own. The main `FastAPI()` app assembles the final application by including one or more routers.
- `app.include_router(...)` — registers all of a router's endpoints onto the main app.
- `prefix="/predictions"` — automatically prepends this path to every route in that router (so `@router.post("/")` inside `predictions.py` becomes `POST /predictions/` on the running app) — you don't repeat the prefix in every route definition.
- `tags=["predictions"]` — groups these endpoints together in the auto-generated Swagger docs, purely for documentation organization.
- Routers can also carry **shared dependencies** applied to every route inside them:
  ```python
  router = APIRouter(dependencies=[Depends(get_current_user)])
  ```
  — e.g. requiring authentication for an entire group of endpoints in one line, instead of adding `Depends(get_current_user)` to every individual route.
- Routers can even be **nested** — one `APIRouter` can `include_router()` another, useful for deeply organized APIs (e.g. `/api/v1/predictions`, `/api/v2/predictions`).

**Why it matters for AI services specifically:** as a service grows past a handful of endpoints (predictions, chat, users, health, admin), `APIRouter` is what keeps each concern in its own file — directly solving the "everything crammed in `main.py`" mistake called out in §20.

---

## 14. Authentication vs Authorization

- **Authentication — "Who are you?"** Verifying identity, usually via a JWT, API key, or session token attached to the request.
- **Authorization — "What can you do?"** Checking permissions — can this authenticated user call this model or see this data?

```python
@app.get("/models/{model_id}")
def get_model(model_id: int, user: User = Depends(get_current_user)):
    if not user.can_access(model_id):
        raise HTTPException(403, "Not authorized for this model")
```

- Flow: `Client → JWT → FastAPI → Authentication → Authorization → Endpoint`. Identity is checked first, permission second.

---

## 15. Error Handling

```python
from fastapi import HTTPException

if model_not_found:
    raise HTTPException(status_code=404, detail="Model not found")
```

- **4xx (client error)** — bad input, missing auth, not found. Safe to describe in detail.

```python
@app.exception_handler(ModelInferenceError)
async def inference_error_handler(request, exc):
    logger.error(f"Inference failed: {exc}")
    return JSONResponse(status_code=500, content={"detail": "Model unavailable"})
```

- **5xx (server error)** — your bug, a crashed model, a timeout. Log the details internally, but **never expose internals** (stack traces, exception messages) directly to the client.

---

## 16. WebSockets — Persistent, Two-Way Connection

```python
@app.websocket("/ws/chat")
async def chat_ws(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            msg = await websocket.receive_text()
            reply = await generate_reply(msg)
            await websocket.send_text(reply)
    except WebSocketDisconnect:
        pass
```

- Unlike plain HTTP (request → response → connection ends), a WebSocket keeps a persistent, bidirectional connection open. Use for real-time chat/voice AI, live inference, or an agent streaming intermediate steps — anywhere **both sides** need to send messages at any time.

---

## 17. Server-Sent Events (SSE) — One-Way Streaming

```python
from fastapi.responses import StreamingResponse

async def token_stream(prompt: str):
    async for token in llm.stream(prompt):
        yield f"data: {token}\n\n"

@app.get("/chat/stream")
async def chat_stream(prompt: str):
    return StreamingResponse(token_stream(prompt), media_type="text/event-stream")
```

- A long-lived HTTP response the server pushes chunks into over time — simpler than a WebSocket, works over plain HTTP.
- **When SSE beats WebSocket:** when you only need server → client streaming (e.g. token-by-token LLM output, progress updates for a long job) and the client never needs to send anything mid-stream.

---

## 18. FastAPI + vLLM Integration

**Why put FastAPI in front of vLLM** (which already exposes an OpenAI-compatible inference server) rather than calling it directly?

- **Authentication & authorization** — vLLM's raw server has no concept of your users or tenants.
- **Validation** — enforce your own request shape before it reaches the model.
- **Rate limiting** — protect GPU capacity from any single caller.
- **Conversation management** — store history, apply system prompts, manage context windows.
- **Business logic** — RAG retrieval, tool calling, post-processing the model's output.
- **Separation of concerns** — vLLM stays focused purely on fast inference.

```python
import httpx
from fastapi import APIRouter

router = APIRouter()
VLLM_URL = "http://vllm-server:8000/v1/chat/completions"

@router.post("/chat")
async def chat(request: ChatRequest):
    payload = {"model": "llama-3-8b", "messages": [{"role": "user", "content": request.message}]}
    async with httpx.AsyncClient(timeout=30.0) as client:
        response = await client.post(VLLM_URL, json=payload)
        response.raise_for_status()
        data = response.json()
        reply = data["choices"][0]["message"]["content"]
    return ChatResponse(reply=reply)
```

- Note the async `httpx` client with an **explicit timeout** — never call an external inference service without one.

```python
async def stream_from_vllm(message: str):
    async with httpx.AsyncClient(timeout=None) as client:
        async with client.stream("POST", VLLM_URL, json={..., "stream": True}) as response:
            async for line in response.aiter_lines():
                if line.startswith("data:"):
                    yield f"{line}\n\n"

@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    return StreamingResponse(stream_from_vllm(request.message), media_type="text/event-stream")
```

- Same SSE pattern (§17) works to relay vLLM's own token stream out to your client. The same shape also works over a WebSocket instead, when the client needs to interrupt generation mid-stream.

---

## 19. Production Considerations

| Concern       | What to watch for                                        |
| ------------- | -------------------------------------------------------- |
| Config        | Env vars, secrets managers — no hardcoded values         |
| Logging       | Structured logs + request IDs for tracing                |
| Health checks | Liveness vs. readiness endpoints                         |
| Metrics       | Prometheus + Grafana for latency & error rates           |
| Scaling       | Multiple workers, horizontal scaling behind a proxy      |
| Resilience    | Timeouts, retries, graceful shutdown, connection pooling |

---

## 20. Common FastAPI Mistakes (And Fixes)

| ✕ Mistake                         | ✓ Fix                                                                  |
| --------------------------------- | ---------------------------------------------------------------------- |
| Everything crammed in `main.py`   | Split into routes/services/repositories as the project grows (§7, §13) |
| Blocking calls inside `async def` | Use `run_in_threadpool` or a dedicated worker for CPU-heavy work (§10) |
| Reloading the model every request | Load once via a dependency or app startup event (§9)                   |
| No input validation               | Let Pydantic define and enforce every request schema (§5)              |
| Returning raw DB objects          | Always use a `response_model` to control the data contract (§6)        |
| No timeout on external calls      | Set explicit timeouts on every HTTP client call to an LLM/API (§18)    |
| No rate limiting on LLM endpoints | Protect expensive inference with per-user/IP limits (§11)              |
| Hardcoded secrets                 | Load config and secrets from environment variables (§19)               |
| No health endpoint / logging      | Add `/health` and structured logs before the first deploy (§19)        |

---

## The Complete Picture

```
Client
  ↓
FastAPI — Validation → Authentication → Rate Limiting
  ↓
Business Logic
  ↓
PostgreSQL   /   vLLM → GPU
  ↓
Response
```

## Quick Reference — Cheat Sheet

| Task                      | Command                                                               |
| ------------------------- | --------------------------------------------------------------------- |
| Define a route            | `@app.get("/path")` / `@router.post("/path")`                         |
| Path parameter            | `def fn(id: int):` with `{id}` in the route path                      |
| Query parameter           | `def fn(limit: int = 10):`                                            |
| Request body              | Pydantic `BaseModel` as a typed function argument                     |
| Response contract         | `@app.post(..., response_model=MyResponse)`                           |
| Inject a dependency       | `param=Depends(get_something)`                                        |
| Group routes into a file  | `router = APIRouter()`, then `app.include_router(router, prefix=...)` |
| Background/thread offload | `await run_in_threadpool(fn, args)`                                   |
| Raise an HTTP error       | `raise HTTPException(status_code, detail)`                            |
| Custom exception handler  | `@app.exception_handler(SomeError)`                                   |
| WebSocket endpoint        | `@app.websocket("/ws/...")`                                           |
| SSE streaming             | `StreamingResponse(generator, media_type="text/event-stream")`        |
| Middleware                | `@app.middleware("http")`                                             |
