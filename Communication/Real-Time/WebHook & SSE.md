# System Design: Webhooks & Server-Sent Events (SSE)

---

# Part 1: Webhooks

## What is a Webhook?

A way for one system (provider) to **notify another system (receiver) in real time** when an event happens, using an HTTP POST request.

Instead of your app asking "did something happen?" every few seconds (polling), the provider **pushes data to you the moment it occurs**.

> Analogy: You don't stand at a restaurant counter asking "is my table ready?" every 2 minutes. You leave your number and they text you when it's ready. That's a webhook.

---

## 1. How Webhooks Work

**Three phases: Registration → Trigger → Delivery**

```
Step 1: You register a webhook URL with the provider
        e.g. GitHub → Settings → Webhooks → https://myapp.com/webhook

Step 2: Provider monitors its internal events
        (PR opened, payment succeeded, file uploaded...)

Step 3: Event fires → provider sends HTTP POST to your URL
        with a JSON payload describing what happened

Step 4: Your server processes the event, returns 200 OK
```

**Real example — Stripe payment succeeded:**
```
Stripe → POST https://yourapp.com/webhook
{
  "id": "evt_1234",
  "type": "payment_intent.succeeded",
  "data": {
    "object": {
      "amount": 4999,
      "currency": "inr",
      "customer": "cus_abc"
    }
  }
}
```

---

## 2. Anatomy of a Webhook Request

| Part | Details |
|---|---|
| **Method** | Always `POST` (body carries structured data) |
| `Content-Type` | `application/json` |
| `User-Agent` | Identifies sender — e.g. `Stripe/1.0`, `GitHub-Hookshot` |
| `X-Event-Type` | What happened — e.g. `payment_intent.succeeded` |
| `X-Signature` | HMAC-SHA256 of payload using shared secret — for verification |
| `X-Request-ID` | Unique delivery ID — use for deduplication and logging |

---

## 3. Setting Up a Webhook Receiver

### Basic FastAPI Receiver

```python
from fastapi import FastAPI, Request, HTTPException, Header
import hmac
import hashlib

app = FastAPI()

WEBHOOK_SECRET = "whsec_your_secret_here"

# Track processed event IDs to ensure idempotency
processed_events: set[str] = set()


def verify_signature(payload: bytes, signature: str, secret: str) -> bool:
    """Verify HMAC-SHA256 signature from provider (e.g. Stripe/GitHub)."""
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)


@app.post("/webhook")
async def receive_webhook(
    request: Request,
    x_signature: str = Header(None),    # e.g. Stripe sends X-Signature
    x_request_id: str = Header(None),   # unique delivery ID
):
    raw_body = await request.body()

    # 1. Verify signature — reject forged requests
    if not x_signature or not verify_signature(raw_body, x_signature, WEBHOOK_SECRET):
        raise HTTPException(status_code=401, detail="Invalid signature")

    payload = await request.json()
    event_id = payload.get("id") or x_request_id

    # 2. Idempotency — skip already processed events
    if event_id in processed_events:
        return {"status": "already_processed"}
    processed_events.add(event_id)

    # 3. Return 200 fast — do heavy work in background
    event_type = payload.get("type")
    print(f"Received event: {event_type} [{event_id}]")

    # Dispatch to handlers
    if event_type == "payment_intent.succeeded":
        await handle_payment(payload["data"]["object"])
    elif event_type == "pull_request":
        await handle_pr(payload)

    return {"status": "ok"}


async def handle_payment(data: dict):
    print(f"Payment of {data['amount']} {data['currency']} succeeded!")
    # → mark order paid, send invoice, notify warehouse


async def handle_pr(data: dict):
    print(f"PR opened: {data.get('pull_request', {}).get('title')}")
    # → trigger CI/CD pipeline
```

---

### GitHub-Specific Signature Verification

```python
import hmac, hashlib

def verify_github_signature(payload: bytes, signature_header: str, secret: str) -> bool:
    """GitHub sends: X-Hub-Signature-256: sha256=<hex_digest>"""
    if not signature_header or not signature_header.startswith("sha256="):
        return False
    expected = "sha256=" + hmac.new(
        secret.encode(), payload, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature_header)
```

---

### Security Checklist

| Risk | Fix |
|---|---|
| Forged/spoofed webhooks | Verify HMAC signature on every request |
| Duplicate deliveries | Store + check processed event IDs (idempotency) |
| Slow processing blocking 200 | Enqueue immediately, process in background worker |
| Sensitive data in logs | Never log raw payloads in plaintext |
| Public endpoint abuse | Optionally whitelist provider IP ranges |

---

## 4. Scalable Webhook Infrastructure (Production)

For high volume (thousands/millions of events daily):

```
Provider
   │
   ▼
POST /webhook  (FastAPI — validate + verify only)
   │
   ▼ enqueue immediately → return 200 OK
Message Queue (Kafka / RabbitMQ / AWS SQS)
   │
   ▼
Background Workers (pull from queue)
   │ ├── deduplicate (event ID)
   │ ├── process business logic
   │ ├── retry with exponential backoff on failure
   │ └── Dead Letter Queue (DLQ) after max retries
   │
   ▼
Event Store (PostgreSQL / MongoDB) — raw payload + status + timestamps
   │
   ▼
Observability (Prometheus + Grafana / Datadog)
   — events/hour, success rate, latency, queue depth, DLQ count
```

**FastAPI enqueue-first pattern:**

```python
from fastapi import FastAPI, Request, BackgroundTasks
import asyncio

app = FastAPI()
event_queue: asyncio.Queue = asyncio.Queue()


@app.post("/webhook")
async def receive_webhook(request: Request, background_tasks: BackgroundTasks):
    payload = await request.json()
    # Validate + verify signature here (same as above)
    background_tasks.add_task(process_event, payload)  # non-blocking
    return {"status": "queued"}   # return 200 fast


async def process_event(payload: dict):
    """Heavy processing happens here, off the request path."""
    event_type = payload.get("type")
    # business logic, DB writes, downstream API calls...
    print(f"Processing: {event_type}")
```

**Retry with exponential backoff:**

```python
import asyncio

async def process_with_retry(payload: dict, max_retries: int = 5):
    backoff = 1
    for attempt in range(max_retries):
        try:
            await process_event(payload)
            return  # success
        except Exception as e:
            if attempt == max_retries - 1:
                await send_to_dlq(payload, str(e))  # give up → DLQ
                return
            await asyncio.sleep(backoff)
            backoff = min(backoff * 2, 60)  # cap at 60s


async def send_to_dlq(payload: dict, error: str):
    print(f"DLQ: {payload.get('id')} — {error}")
    # → store in DB, trigger alert
```

---

---

# Part 2: Server-Sent Events (SSE)

## What is SSE?

A web technology where the **server pushes updates to the client over a single persistent HTTP connection**. The client opens the connection once — the server streams data to it whenever new events occur.

> Think of SSE as a **one-way radio broadcast** — server is the station, clients tune in and listen.

**Key characteristics:**
- **Unidirectional** — server → client only (no client-to-server messages)
- **Text-based** — simple `text/event-stream` format over HTTP
- **Persistent** — single connection stays open for multiple messages
- **Auto-reconnecting** — browser's `EventSource` API reconnects automatically
- **Native browser support** — no extra libraries needed in the browser

---

## How SSE Protocol Works

### HTTP Response Format

SSE uses `Content-Type: text/event-stream`. Data is sent as **plain text frames**:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: Hello World\n\n

event: price_update
data: {"symbol": "AAPL", "price": 189.45}\n\n

id: 42
event: alert
data: {"message": "Server maintenance in 5 min"}\n\n

: this is a comment (ignored by client)\n\n
```

**SSE frame fields:**

| Field | Purpose |
|---|---|
| `data:` | The message payload (required) |
| `event:` | Custom event type name (optional) |
| `id:` | Event ID — sent as `Last-Event-ID` on reconnect |
| `retry:` | Tells client how many ms to wait before reconnecting |
| `: ` | Comment — used as heartbeat to keep connection alive |

Multiple `data:` lines are joined with `\n`. A blank line (`\n\n`) ends the event.

---

## The EventSource API (Browser Client)

```javascript
// Browser — native, no library needed
const es = new EventSource('/stream');

// Default message handler
es.onmessage = (event) => {
  console.log('Received:', event.data);
};

// Named event handler
es.addEventListener('price_update', (event) => {
  const data = JSON.parse(event.data);
  updateUI(data.symbol, data.price);
});

// Connection events
es.onopen  = () => console.log('Connected');
es.onerror = () => console.log('Error — browser will auto-reconnect');

// Close when done
es.close();
```

**Auto-reconnection:** If the connection drops, the browser automatically reconnects after `retry` ms (default 3000ms). It sends the last received `id` in the `Last-Event-ID` header so the server can resume from where it left off.

---

## FastAPI SSE Implementation

FastAPI supports SSE via `StreamingResponse` with an async generator.

### Basic SSE Stream

```python
import asyncio
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()


async def event_generator():
    """Yields SSE-formatted strings indefinitely."""
    count = 0
    while True:
        count += 1
        yield f"id: {count}\ndata: Message {count}\n\n"
        await asyncio.sleep(1)  # push every second


@app.get("/stream")
async def stream():
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # disable nginx buffering
        }
    )
```

---

### Live Stock Price Stream

```python
import asyncio
import json
import random
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse

app = FastAPI()

# Simulated stock prices
STOCKS = {"AAPL": 189.0, "GOOGL": 175.0, "MSFT": 420.0}


async def stock_stream(request: Request):
    while True:
        # Check if client disconnected
        if await request.is_disconnected():
            break

        # Simulate price movement
        for symbol in STOCKS:
            STOCKS[symbol] += random.uniform(-1.0, 1.0)
            STOCKS[symbol] = round(STOCKS[symbol], 2)

        payload = json.dumps(STOCKS)
        yield f"event: price_update\ndata: {payload}\n\n"

        await asyncio.sleep(1)


@app.get("/stocks/stream")
async def stream_stocks(request: Request):
    return StreamingResponse(
        stock_stream(request),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"}
    )
```

---

### With Heartbeat (Keep Connection Alive Through Proxies)

```python
import asyncio
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse

app = FastAPI()

# Shared event store — in production use Redis Pub/Sub
events: list[dict] = []
subscribers: list[asyncio.Queue] = []


async def notify_subscribers(event: dict):
    for queue in subscribers:
        await queue.put(event)


@app.post("/publish")
async def publish(event: dict):
    events.append(event)
    await notify_subscribers(event)
    return {"status": "published"}


async def sse_stream(request: Request, last_event_id: int = 0):
    # Send any missed events first (resume after reconnect)
    for event in events:
        if event["id"] > last_event_id:
            yield f"id: {event['id']}\ndata: {json.dumps(event)}\n\n"

    # Subscribe to future events
    queue: asyncio.Queue = asyncio.Queue()
    subscribers.append(queue)

    try:
        while True:
            if await request.is_disconnected():
                break

            try:
                # Wait for new event with heartbeat timeout
                event = await asyncio.wait_for(queue.get(), timeout=20)
                yield f"id: {event['id']}\ndata: {json.dumps(event)}\n\n"
            except asyncio.TimeoutError:
                yield ": heartbeat\n\n"  # keep-alive comment every 20s
    finally:
        subscribers.remove(queue)


@app.get("/events/stream")
async def stream_events(request: Request, last_event_id: int = 0):
    import json
    return StreamingResponse(
        sse_stream(request, last_event_id),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"}
    )
```

---

## SSE vs WebSockets: When to Use Which

| | SSE | WebSocket |
|---|---|---|
| **Direction** | Server → Client only | ✅ Bidirectional |
| **Protocol** | Plain HTTP | Custom WS protocol over TCP |
| **Browser support** | ✅ Native `EventSource` | ✅ Native |
| **Auto-reconnect** | ✅ Built-in | ❌ Must implement yourself |
| **Proxy/firewall** | ✅ Works everywhere (HTTP) | ⚠️ Sometimes blocked |
| **Load balancing** | ✅ Standard HTTP LB | ⚠️ Sticky sessions needed |
| **Complexity** | Low | Medium-High |
| **Binary data** | ❌ Text only | ✅ Binary + text |

**Use SSE when:** Live feeds, notifications, dashboards, AI streaming responses — anything server-pushes, client-reads.

**Use WebSockets when:** Chat, multiplayer games, collaborative editing — anything needing two-way real-time communication.

---

## Key Takeaways

**Webhooks:**
- Event-driven HTTP callbacks — provider pushes to your URL when something happens
- Always verify HMAC signature, implement idempotency (deduplicate by event ID)
- Return 200 fast — enqueue event, process asynchronously in background workers
- Use DLQ + retry with exponential backoff for reliability

**SSE:**
- One-way server-push over plain HTTP — perfect for feeds, dashboards, AI streaming
- `text/event-stream` format with `data:`, `event:`, `id:`, `retry:` fields
- Browser `EventSource` auto-reconnects and sends `Last-Event-ID` header
- Send heartbeat comments (`: ping\n\n`) every 15–20s to survive proxy timeouts
- Simpler and more HTTP-friendly than WebSockets for unidirectional use cases

---
