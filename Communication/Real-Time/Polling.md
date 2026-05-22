# System Design: Long Polling

---

## What is Long Polling?

A technique where the **client makes a request and the server holds it open** until new data is available (or a timeout occurs), then responds. The client immediately sends another request — creating a near-real-time update loop without a persistent connection.

> Long polling sits between basic polling and true real-time (WebSockets) — it's the pragmatic middle ground.

---

## Long Polling vs Short Polling

### Short Polling
Client repeatedly hammers the server on a fixed interval regardless of whether anything changed.

```
Client:  GET /updates  →  Server: "nothing new" (200)
Client:  GET /updates  →  Server: "nothing new" (200)   ← wasted requests
Client:  GET /updates  →  Server: "here's new data" (200)
(repeat every N seconds)
```

### Long Polling
Client sends a request → server **waits silently** until data is ready → responds → client immediately re-connects.

```
Client:  GET /updates  ─────────────────────────────► Server holds...
                                                        (data arrives)
Client:  ◄──────────────────────── "here's new data" ─ Server responds
Client:  GET /updates  ─────────────────────────────► Server holds again...
```

| | Short Polling | Long Polling |
|---|---|---|
| **Connection** | New request every N seconds | Held open until data or timeout |
| **Wasted requests** | ❌ Many | ✅ Minimal |
| **Latency** | Up to N seconds delay | Near-instant on data arrival |
| **Server load** | High (constant requests) | Lower (fewer round trips) |
| **Implementation** | Very simple | Moderate |
| **Infrastructure** | Standard HTTP | Standard HTTP |

---

## How Long Polling Works

**Step-by-step:**

```
1. Client sends GET /poll?last_id=42
2. Server checks — no new data yet
3. Server HOLDS the connection open (suspends the handler)
4. New data arrives on the server (DB write, event, message)
5. Server responds immediately with the new data
6. Client processes response → fires next GET /poll?last_id=43
7. If nothing arrives within timeout (e.g. 30s) → server returns 204/empty
8. Client reconnects immediately on timeout
```

**Key design decisions:**
- **Timeout value** — typically 20–60 seconds; prevents proxy/firewall from killing idle connections
- **Last event ID** — client sends the ID of the last message received so server knows what's new
- **Immediate reconnect** — client must reconnect right after receiving a response or timeout

---

## Implementation Patterns

### Pattern 1: Simple Queue-Based Long Poll

Each client waits on an `asyncio.Event` (or queue). When data arrives, the server signals all waiting clients.

**FastAPI Server:**
```python
import asyncio
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Dict, List

app = FastAPI()

# Stores the actual messages: { "user_id": ["msg1", "msg2"] }
message_store: Dict[str, List[str]] = {}

# Stores an asyncio.Event for each user: { "user_id": asyncio.Event }
event_store: Dict[str, asyncio.Event] = {}

class Message(BaseModel):
    user_id: str
    content: str


@app.post("/send-message")
async def send_message(message: Message):
    # 1. Save the message
    if message.user_id not in message_store:
        message_store[message.user_id] = []
    message_store[message.user_id].append(message.content)
    
    # 2. If there is a client waiting, wake them up immediately
    if message.user_id in event_store:
        event_store[message.user_id].set()  # This triggers the Event!
        
    return {"status": "Message sent"}


@app.get("/poll")
async def poll_messages(user_id: str, timeout: int = 30):
    # If messages already exist, return them immediately without waiting
    if user_id in message_store and message_store[user_id]:
        return {"messages": reset_and_get_messages(user_id)}

    # If no messages, create an event for this user if it doesn't exist
    if user_id not in event_store:
        event_store[user_id] = asyncio.Event()
    
    user_event = event_store[user_id]

    try:
        # This is the magic line. The server pauses here completely.
        # It consumes 0% CPU while waiting. It will wake up ONLY if:
        # a) The event is .set() by /send-message, OR b) The timeout expires.
        await asyncio.wait_for(user_event.wait(), timeout=timeout)
    except asyncio.TimeoutError:
        # The timeout hit, and no message arrived
        return {"messages": [], "reason": "timeout"}
    finally:
        # Clean up the event object so we don't leak memory
        event_store.pop(user_id, None)

    # If we reached here, the event was triggered! Get the data.
    return {"messages": reset_and_get_messages(user_id)}


def reset_and_get_messages(user_id: str) -> List[str]:
    """Helper to safely extract messages and clear the queue."""
    messages = message_store[user_id].copy()
    message_store[user_id].clear()
    return messages
**Python Client:**

```python
import httpx
import asyncio

async def poll_for_updates():
    last_id = 0
    async with httpx.AsyncClient(timeout=40) as client:
        while True:
            try:
                response = await client.get(
                    "http://localhost:8000/poll",
                    params={"last_id": last_id, "timeout": 30}
                )
                if response.status_code == 200:
                    data = response.json()
                    for msg in data.get("messages", []):
                        print(f"New message [{msg['id']}]: {msg['text']}")
                        last_id = msg["id"]  # advance cursor
                # 204 = timeout, no new data → reconnect immediately
            except httpx.ReadTimeout:
                pass  # reconnect immediately
            except Exception as e:
                print(f"Connection error: {e}")
                await asyncio.sleep(2)  # brief backoff on error

asyncio.run(poll_for_updates())
```

---

### Pattern 2: Notification-Style Long Poll (e.g. Chat App)

Per-user queues — each client only receives messages intended for them.

```python
import asyncio
from fastapi import FastAPI, Path
from fastapi.responses import JSONResponse
from collections import defaultdict

app = FastAPI()

# Per-user async queues
user_queues: dict[str, asyncio.Queue] = defaultdict(asyncio.Queue)


@app.post("/message/{user_id}")
async def send_to_user(user_id: str, text: str):
    """Push a message into a specific user's queue."""
    await user_queues[user_id].put({"text": text})
    return {"status": "queued"}


@app.get("/poll/{user_id}")
async def poll_for_user(user_id: str, timeout: int = 30):
    """
    Long-poll endpoint for a specific user.
    Holds until a message arrives in their queue or timeout.
    """
    queue = user_queues[user_id]
    try:
        message = await asyncio.wait_for(queue.get(), timeout=timeout)
        return JSONResponse({"message": message})
    except asyncio.TimeoutError:
        return JSONResponse({"message": None}, status_code=204)
```

---

## Handling Challenges

### 1. Connection Timeouts (Proxies / Firewalls)
Intermediate proxies often kill idle connections after 60–90 seconds.

**Fix:** Set server timeout to 20–30s and always return before proxy kills it. Client reconnects immediately.

```python
# Always timeout before the proxy does
await asyncio.wait_for(event.wait(), timeout=25)  # safe under 30s proxy limit
```

---

### 2. Missed Messages (Client Reconnect Gap)
Between the server responding and the client reconnecting, messages can be missed.

**Fix:** Use a **cursor/last_id** — client tracks the last message ID and sends it on every request. Server returns everything after that ID.

```python
# Client always sends its last known position
GET /poll?last_id=42&timeout=30

# Server returns all messages with id > 42
new_messages = [m for m in messages if m["id"] > last_id]
```

---

### 3. Server Resource Usage (Many Concurrent Connections)
Each held connection occupies a coroutine. With 10,000 users, that's 10,000 waiting coroutines.

**Fix:** Use `asyncio` (non-blocking) — waiting coroutines are lightweight (few KB each vs threads at ~1MB each). For production, use Redis Pub/Sub to signal waiters across multiple server instances.

```python
# Production: use Redis pub/sub instead of in-memory events
import aioredis

redis = aioredis.from_url("redis://localhost")

@app.get("/poll")
async def long_poll(last_id: int = 0, timeout: int = 30):
    pubsub = redis.pubsub()
    await pubsub.subscribe("new_messages")
    try:
        async with asyncio.timeout(timeout):
            async for message in pubsub.listen():
                if message["type"] == "message":
                    new_msgs = [m for m in messages if m["id"] > last_id]
                    return JSONResponse({"messages": new_msgs})
    except TimeoutError:
        return JSONResponse({"messages": []}, status_code=204)
    finally:
        await pubsub.unsubscribe("new_messages")
```

---

### 4. Client Backoff on Errors
If the server is down, the client must not hammer it continuously.

```python
async def poll_with_backoff():
    backoff = 1
    while True:
        try:
            response = await client.get("/poll", params={"last_id": last_id})
            backoff = 1  # reset on success
        except Exception:
            await asyncio.sleep(min(backoff, 30))
            backoff *= 2  # exponential backoff: 1s, 2s, 4s, 8s... up to 30s
```

---

## When to Use Long Polling

| ✅ Good fit | ❌ Not a good fit |
|---|---|
| Notifications (email, alerts) | High-frequency updates (>1/sec) |
| Chat with low-moderate message volume | Bidirectional streaming (use WebSockets) |
| Dashboard status updates | Very large number of concurrent users (WebSockets more efficient) |
| Environments where WebSockets are blocked (corporate proxies) | Binary data streaming |
| Simple infra — no WebSocket support needed | |

---

## Key Takeaways

- Long polling = client holds connection open; server responds only when data is ready
- Use a **cursor (last_id)** to prevent missed messages on reconnect
- Set server **timeout < proxy timeout** (25–30s is safe); client reconnects immediately
- Use **`asyncio`** in Python/FastAPI — waiting connections are coroutines, not threads (scales well)
- For **multi-server** setups, use **Redis Pub/Sub** to signal waiters across instances
- Long polling is a pragmatic choice when WebSockets aren't available or the update frequency is low

---

