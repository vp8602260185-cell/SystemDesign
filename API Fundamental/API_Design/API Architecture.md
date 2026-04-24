# System Design: API Architectural Styles

---

## What is an API Architectural Style?

An **API architectural style** is a set of conventions and constraints that define how systems communicate over a network. It governs:

- How requests and responses are structured
- What protocol is used (HTTP, TCP, WebSocket, etc.)
- How errors are handled
- How the API evolves over time

> Choosing the right API architecture is not about picking the most popular option — it is about matching your requirements to the style that fits best.

---

## 1. SOAP (Simple Object Access Protocol)

A **protocol** (not just a style) that uses **XML** for message formatting and typically runs over HTTP or SMTP.

### How It Works
- Client sends an XML-formatted **envelope** containing a header and body
- Server processes the request and returns an XML response
- Contracts are defined using **WSDL** (Web Services Description Language)

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <GetUser>
      <UserId>101</UserId>
    </GetUser>
  </soap:Body>
</soap:Envelope>
```

### Pros
- Strict contract via WSDL — both sides know exactly what to expect
- Built-in standards for security (WS-Security), transactions, and reliability
- Language and platform agnostic

### Cons
- Very verbose XML payloads
- Complex to implement and maintain
- Overkill for most modern web applications

### Best For
- Enterprise/financial systems (banking, insurance, payment gateways)
- Legacy system integrations
- Environments requiring strict contracts and formal compliance (e.g., healthcare with HL7)

---

## 2. REST (Representational State Transfer)

The dominant architectural style for web APIs today. Built on top of **HTTP** and uses standard methods (GET, POST, PUT, DELETE, PATCH).

### Core Principles (Constraints)
1. **Stateless** — each request contains all info needed; server holds no session state
2. **Client-Server** — concerns are separated between UI and data storage
3. **Uniform Interface** — consistent resource-based URLs
4. **Cacheable** — responses declare whether they can be cached
5. **Layered System** — client doesn't need to know if it's talking to a load balancer or origin server

### How It Works

| HTTP Method | Action | Example |
|---|---|---|
| `GET` | Read | `GET /users/101` |
| `POST` | Create | `POST /users` |
| `PUT` | Full update | `PUT /users/101` |
| `PATCH` | Partial update | `PATCH /users/101` |
| `DELETE` | Delete | `DELETE /users/101` |

### Pros
- Simple and intuitive — maps well to CRUD operations
- Stateless design scales horizontally easily
- Excellent tooling, documentation, and ecosystem
- Human-readable (typically JSON)

### Cons
- **Over-fetching** — response may contain more data than needed
- **Under-fetching** — may need multiple requests to get related data
- No strict contract (unless OpenAPI/Swagger is used)
- Not ideal for real-time or streaming use cases

### Best For
- Public-facing APIs and mobile backends
- CRUD-heavy services
- Microservices that communicate over HTTP

---

## 3. GraphQL

Developed by **Facebook (Meta)** in 2012, open-sourced in 2015. A **query language for APIs** — clients request exactly the data they need.

### How It Works
- Single endpoint (typically `POST /graphql`)
- Client sends a **query** describing the exact shape of data it wants
- Server returns exactly that — no more, no less

```graphql
query {
  user(id: "101") {
    name
    email
    posts {
      title
      createdAt
    }
  }
}
```

### Operations
| Type | Purpose |
|---|---|
| `query` | Read data |
| `mutation` | Write / update data |
| `subscription` | Real-time updates over WebSocket |

### Pros
- Eliminates over-fetching and under-fetching
- Single request can fetch deeply nested, related data
- Strongly typed schema (`SDL`) acts as living documentation
- Great for complex, interconnected data models

### Cons
- More complex to implement on the server side
- Caching is harder (everything hits one endpoint)
- File uploads require workarounds
- N+1 query problem (requires DataLoader or similar)

### Best For
- Mobile apps (minimize data usage)
- Complex frontends with varied data needs (e.g., dashboards)
- Aggregating data from multiple services (BFF pattern)
- Rapid product iteration where data requirements change frequently

---

## 4. gRPC (Google Remote Procedure Call)

A high-performance, open-source RPC framework developed by **Google**. Uses **Protocol Buffers (Protobuf)** as its IDL and data format, runs over **HTTP/2**.

### How It Works
- Define services and messages in a `.proto` file
- Code is auto-generated for client and server in any supported language
- Client calls remote methods as if they were local functions

```proto
service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
}

message UserRequest {
  int32 user_id = 1;
}

message UserResponse {
  string name = 1;
  string email = 2;
}
```

### Communication Patterns
| Pattern | Description |
|---|---|
| Unary | Single request → single response |
| Server streaming | Single request → stream of responses |
| Client streaming | Stream of requests → single response |
| Bidirectional streaming | Stream of requests ↔ stream of responses |

### Pros
- Extremely fast — binary Protobuf + HTTP/2 multiplexing
- Strongly typed contracts auto-generate client/server code
- Native streaming support
- Excellent for polyglot environments

### Cons
- Not human-readable (binary format)
- Limited browser support (requires gRPC-Web proxy)
- Steeper learning curve than REST
- Harder to debug without tooling

### Best For
- High-performance internal microservice communication
- Real-time streaming between services
- Polyglot systems needing cross-language RPC
- Mobile clients needing low-latency, low-bandwidth APIs

---

## 5. WebSocket

A **full-duplex, bidirectional communication protocol** over a single persistent TCP connection. Established via an HTTP upgrade handshake.

### How It Works
1. Client sends an HTTP request with `Upgrade: websocket` header
2. Server responds `101 Switching Protocols`
3. A persistent TCP connection is established
4. Both sides can send messages at any time without polling

```
Client ──── HTTP Upgrade ────► Server
Client ◄─── 101 Switching ──── Server
Client ◄──── Message ─────────► Server  (bidirectional, persistent)
```

### Pros
- True real-time, low-latency communication
- No repeated HTTP overhead per message
- Server can push data to client without client requesting it
- Efficient for high-frequency message exchange

### Cons
- Stateful — harder to scale horizontally (needs sticky sessions or pub/sub broker)
- Not automatically resumable on disconnect
- More complex error handling and reconnection logic
- Overkill for infrequent updates

### Best For
- Live chat and messaging apps
- Multiplayer games
- Real-time dashboards and stock tickers
- Collaborative editing tools (e.g., Google Docs)
- Live sports/financial data feeds

---

## 6. Webhook

An **event-driven, HTTP callback** mechanism. Instead of the client polling the server, the server pushes a notification to a pre-registered URL when an event occurs.

### How It Works
1. Client registers a callback URL with the server
2. When an event occurs, the server sends an HTTP `POST` to that URL
3. Client's endpoint receives and processes the payload

```
Event occurs on Server
       │
       ▼
POST https://your-app.com/webhook
{
  "event": "payment.succeeded",
  "amount": 4999,
  "userId": "101"
}
```

### Pros
- No polling — efficient, event-driven
- Simple to implement on both sides
- Works over plain HTTP — no special protocol needed
- Great for asynchronous workflows

### Cons
- Client must expose a public HTTPS endpoint
- Delivery not guaranteed — need retry logic and idempotency
- Hard to test locally (requires tunneling tools like ngrok)
- Ordering and deduplication are the consumer's responsibility

### Best For
- Payment notifications (Stripe, Razorpay)
- CI/CD pipeline triggers (GitHub webhooks)
- Third-party integrations and automation workflows
- Any async notification where the client doesn't need to poll

---

## Summary

| Style | Protocol | Format | Communication | Best For |
|---|---|---|---|---|
| **SOAP** | HTTP/SMTP | XML | Request-Response | Enterprise, legacy, strict contracts |
| **REST** | HTTP | JSON/XML | Request-Response | Public APIs, CRUD, web/mobile backends |
| **GraphQL** | HTTP | JSON | Request-Response | Complex/nested data, mobile, BFF |
| **gRPC** | HTTP/2 | Protobuf (binary) | Unary + Streaming | Microservices, high-performance RPC |
| **WebSocket** | TCP | Any (JSON, binary) | Bidirectional persistent | Real-time apps, chat, gaming |
| **Webhook** | HTTP | JSON | Event-driven push | Async notifications, integrations |

---

## Quick Decision Guide

```
Do you need real-time bidirectional communication?
└── YES → WebSocket

Do you want event-driven async notifications?
└── YES → Webhook

Are you building high-performance internal microservices?
└── YES → gRPC

Do you have complex, nested data with varied client needs?
└── YES → GraphQL

Is it a public API or standard CRUD service?
└── YES → REST

Is it a legacy enterprise integration requiring strict contracts?
└── YES → SOAP
```

---