# System Design: API Gateways

## What is an API Gateway?

A **central server** that sits between clients (browsers, mobile apps) and backend microservices. Instead of clients talking to many services directly, all requests go through the gateway — which handles routing, security, and other operational concerns.

---

## 1. Why Do We Need One?

In a microservices architecture (e.g., e-commerce with separate User, Payment, and Inventory services):

| Without Gateway | With Gateway |
|---|---|
| Clients must know every service's location | Single entry point for all clients |
| Auth & rate limiting duplicated per service | Centralised cross-cutting concerns |
| Hard to enforce consistent security | Uniform access control |

---

## 2. Core Features

| Feature | What It Does |
|---|---|
| **Auth & Authorization** | Validates JWT/OAuth tokens or API keys; checks permissions centrally so individual services don't have to |
| **Rate Limiting** | Caps requests per client per window (e.g. 100 req/min); returns `429` if exceeded; guards against DoS |
| **Load Balancing** | Distributes traffic across healthy service instances using round-robin, least-connections, etc. |
| **Caching** | Stores responses to frequent requests (e.g. product catalogs) to cut latency and backend load |
| **Request Transformation** | Converts request/response formats between client and backend (e.g. XML → JSON for legacy services) |
| **Service Discovery** | Dynamically finds the right backend instance even as services scale up/down |
| **Circuit Breaking** | Stops forwarding requests to a failing/slow service to prevent cascading failures |
| **Logging & Monitoring** | Logs request metadata, collects latency/error metrics; integrates with Prometheus, Grafana, CloudWatch |

---

## 3. How It Works (Step-by-Step)

Using a food delivery "Place Order" request as an example:

1. **Request Reception** — Gateway receives the request as the sole entry point
2. **Validation** — Checks required fields, correct format, expected schema; rejects malformed requests immediately
3. **Auth** — Verifies identity via token/identity provider; returns `401`/`403` on failure
4. **Rate Limiting** — Checks request frequency; returns `429 Too Many Requests` if over limit
5. **Request Transformation** — Converts data as needed (e.g. plain-text address → GPS coordinates)
6. **Routing** — Uses service discovery + load balancing to route to the correct healthy service instances (Order, Inventory, Payment, Delivery)
7. **Response Handling** — Transforms response format for the client; optionally caches result
8. **Logging** — Records metrics (source, destination, latency, errors) throughout the entire lifecycle

---

## Key Takeaway

> The API Gateway is the **single front door** to your system. It decouples clients from backend complexity and centralises cross-cutting concerns — auth, rate limiting, caching, observability — so individual services stay lean and focused on business logic.

