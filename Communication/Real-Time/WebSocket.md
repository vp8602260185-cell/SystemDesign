# System Design: WebSockets


---

## What is a WebSocket?

A **persistent, full-duplex communication channel** over a single TCP connection. Unlike HTTP where the client always initiates, WebSockets let **both client and server send messages at any time** after the connection is established.

---

## 1. How Do WebSockets Work?

### The Handshake (HTTP → WebSocket Upgrade)

WebSockets start as a normal HTTP request then upgrade:

```
Client → Server:
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server → Client:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After `101 Switching Protocols` — the TCP connection stays open and both sides speak the **WebSocket frame protocol**, not HTTP anymore.

### Message Flow After Handshake

```
Client                        Server
  │                              │
  │──── connect (WS upgrade) ───►│
  │◄─── 101 Switching Protocols ─│
  │                              │  ← persistent TCP connection open
  │──── "Hello" ────────────────►│
  │◄─── "Hi there!" ─────────────│
  │◄─── "New notification" ───── │  ← server pushes without client asking
  │──── "ping" ─────────────────►│
  │◄─── "pong" ──────────────────│
  │──── close frame ────────────►│
  │◄─── close frame ─────────────│
```

---

## 2. Why Are WebSockets Used?

| Problem with HTTP | WebSocket Solution |
|---|---|
| Client must always initiate — server can't push | Server can push anytime |
| Every request has headers (~500 bytes overhead) | Framed messages — minimal overhead after handshake |
| New TCP connection per request (without HTTP/2) | Single persistent TCP connection reused |
| Polling wastes bandwidth | Event-driven — only send when there's data |

**Result:** Ultra-low latency, bidirectional, efficient real-time communication.

---

## 3. WebSockets vs HTTP, Polling, and Long Polling

| | Short Polling | Long Polling | HTTP/2 SSE | WebSocket |
|---|---|---|---|---|
| **Direction** | Client→Server | Client→Server | Server→Client only | ✅ Both ways |
| **Connection** | New per request | Held open | Persistent (one-way) | Persistent (two-way) |
| **Latency** | Up to N seconds | Near real-time | Near real-time | ✅ Real-time |
| **Overhead** | High (headers each time) | Medium | Low | ✅ Very low |
| **Server push** | ❌ No | ❌ No | ✅ Yes | ✅ Yes |
| **Complexity** | Very low | Low | Low | Medium |
| **Proxy/firewall** | ✅ Easy | ✅ Easy | ✅ Easy | ⚠️ Sometimes blocked |

---

## 4. Challenges and Considerations

### 1. Scalability — Sticky Sessions or Pub/Sub

Each WebSocket connection is **stateful** and tied to a specific server instance. If you have 3 servers, clients connected to Server A can't receive messages sent from Server B.

**Fix:** Use Redis Pub/Sub as a message broker — all servers subscribe to the same channel and relay messages to their local connected clients.

```
Client A ──► Server 1 ──► Redis Pub/Sub ──► Server 2 ──► Client B
```

---

### 2. Connection Drops & Reconnection

Network hiccups, mobile switching between WiFi/4G, server restarts — connections drop.

**Fix:** Client-side auto-reconnect with exponential backoff:

```python
# Client reconnects with backoff on disconnect
backoff = 1
while True:
    try:
        async with websockets.connect(url) as ws:
            backoff = 1  # reset on successful connection
            async for message in ws:
                handle(message)
    except Exception:
        await asyncio.sleep(min(backoff, 30))
        backoff *= 2
```

---

### 3. Heartbeat / Ping-Pong

Idle connections can be killed by load balancers or proxies after a timeout.

**Fix:** Send periodic ping frames. WebSocket protocol has built-in ping/pong frames.

```python
# Server sends ping every 30s; client must pong back
# FastAPI/Starlette handles ping/pong automatically via websocket.receive()
```

---

### 4. Authentication

WebSocket connections can't use standard `Authorization` headers after the upgrade. Options:

- Pass token as a **query parameter** during handshake: `ws://api.com/ws?token=JWT`
- Send auth as the **first message** after connecting
- Use a **cookie** (sent automatically by browser on upgrade request)

---

### 5. Message Ordering & At-Least-Once Delivery

TCP guarantees ordering within a connection but not across reconnects.

**Fix:** Add sequence numbers to messages; client requests missed messages on reconnect.

---

## 5. Where Are WebSockets Used?

| Use Case | Why WebSocket |
|---|---|
| **Chat apps** (WhatsApp Web, Slack) | Bidirectional, real-time messages |
| **Multiplayer games** | Ultra-low latency, frequent state updates |
| **Live dashboards** | Server pushes metric updates instantly |
| **Collaborative editing** (Google Docs) | Every keystroke needs to sync bidirectionally |
| **Financial tickers** (stock prices) | Server streams price updates continuously |
| **Live notifications** | Server pushes alerts without client polling |

---

## 6. Implementing WebSockets in FastAPI

FastAPI has **native WebSocket support** via `fastapi.WebSocket` — no extra library needed.

---

### Basic Echo Server

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()          # complete the handshake
    try:
        while True:
            data = await websocket.receive_text()        # wait for client message
            await websocket.send_text(f"Echo: {data}")  # send back
    except Exception:
        pass  # client disconnected
```

---

### Multi-Client Broadcast (Chat Room)

A `ConnectionManager` tracks all active connections and broadcasts to all of them.

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List

app = FastAPI()


class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

    async def send_personal(self, message: str, websocket: WebSocket):
        await websocket.send_text(message)


manager = ConnectionManager()


@app.websocket("/ws/{username}")
async def chat_endpoint(websocket: WebSocket, username: str):
    await manager.connect(websocket)
    await manager.broadcast(f"🟢 {username} joined the chat")
    try:
        while True:
            message = await websocket.receive_text()
            await manager.broadcast(f"{username}: {message}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast(f"🔴 {username} left the chat")
```

---

### With Authentication (Token via Query Param)

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Query, HTTPException
import jwt  # pip install PyJWT

app = FastAPI()
SECRET_KEY = "your-secret-key"


def verify_token(token: str) -> dict:
    try:
        return jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")


@app.websocket("/ws/secure")
async def secure_ws(
    websocket: WebSocket,
    token: str = Query(...)  # ws://host/ws/secure?token=JWT
):
    try:
        payload = verify_token(token)
        user_id = payload["sub"]
    except HTTPException:
        await websocket.close(code=1008)  # Policy Violation
        return

    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"[{user_id}] You sent: {data}")
    except WebSocketDisconnect:
        print(f"User {user_id} disconnected")
```

---

### Scaled Multi-Server with Redis Pub/Sub

```python
import asyncio
import json
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
import aioredis

app = FastAPI()
CHANNEL = "chat"

# In-memory connections on THIS server instance
local_connections: list[WebSocket] = []


async def redis_listener():
    """Listen to Redis channel and broadcast to local clients."""
    redis = aioredis.from_url("redis://localhost")
    pubsub = redis.pubsub()
    await pubsub.subscribe(CHANNEL)
    async for message in pubsub.listen():
        if message["type"] == "message":
            text = message["data"].decode()
            for ws in local_connections.copy():
                try:
                    await ws.send_text(text)
                except Exception:
                    local_connections.remove(ws)


@app.on_event("startup")
async def startup():
    asyncio.create_task(redis_listener())  # start listener in background


@app.websocket("/ws/{username}")
async def ws_endpoint(websocket: WebSocket, username: str):
    await websocket.accept()
    local_connections.append(websocket)
    redis = aioredis.from_url("redis://localhost")

    try:
        while True:
            message = await websocket.receive_text()
            payload = json.dumps({"user": username, "text": message})
            await redis.publish(CHANNEL, payload)  # publish to all servers
    except WebSocketDisconnect:
        local_connections.remove(websocket)
        await redis.publish(CHANNEL, json.dumps({
            "user": "system",
            "text": f"{username} left"
        }))
```

---

### Python WebSocket Client

```python
import asyncio
import websockets  # pip install websockets

async def chat_client():
    uri = "ws://localhost:8000/ws/Alice"
    backoff = 1

    while True:
        try:
            async with websockets.connect(uri) as ws:
                backoff = 1  # reset on success
                print("Connected!")

                # Send and receive concurrently
                async def sender():
                    while True:
                        msg = input()
                        await ws.send(msg)

                async def receiver():
                    async for message in ws:
                        print(f"Received: {message}")

                await asyncio.gather(sender(), receiver())

        except Exception as e:
            print(f"Disconnected: {e}. Retrying in {backoff}s...")
            await asyncio.sleep(backoff)
            backoff = min(backoff * 2, 30)  # exponential backoff cap at 30s

asyncio.run(chat_client())
```

---

## 7. Conclusion — Key Takeaways

- WebSocket = persistent TCP connection, full-duplex, both sides push anytime
- Starts as HTTP, upgrades via `101 Switching Protocols`
- Use for **chat, games, live dashboards, collaborative tools, financial tickers**
- FastAPI has **native WebSocket support** — `async def` route with `WebSocket` param
- For **multi-server scale** → Redis Pub/Sub as message relay between instances
- Handle **auth** via query token, first message, or cookie
- Always handle `WebSocketDisconnect` and implement **client-side reconnect with backoff**
- Send **heartbeat pings** to keep connections alive through proxies/load balancers

---
